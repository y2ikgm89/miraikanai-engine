# Miraikanai Engine repository guidance

## Scope

- This file applies to the entire repository. A closer `AGENTS.md` or
  `AGENTS.override.md` may add or override rules for its subtree.
- This repository currently contains Architecture documents and Codex
  configuration, not Engine implementation, executable Schema, Registry,
  Fixture, Receipt, Build system, or CI.
- Treat every Architecture Owner document according to its header. At the
  current repository state, a `review` document or an exact-looking type,
  value, hash, Registry, Operation, or Fixture is not evidence that an
  implementation or materialized artifact exists.

## Sources of truth

1. Start with `docs/architecture/README.md` for navigation and reading order.
   The index is manually maintained navigation and is not itself normative.
2. Read `docs/architecture/01-governance/architecture-governance.md` before
   changing Architecture ownership, document state, evidence wording,
   dependencies, or ADRs.
3. Locate the one Owner document whose `正本範囲` covers the subject, then read
   its `規範依存` and only the related documents needed for the change.
4. Treat `docs/architecture/decisions/` as decision history. Current contracts,
   fixed values, runtime behavior, and Gates remain in Owner documents.
5. Treat appendices, proposals, reviews, and `docs/superpowers/specs/` according
   to their own headers. Do not use them to silently replace an Owner.

Repository files, headers, manifests, configuration, and committed artifacts
are the source of truth for repository state. For external APIs, standards,
SDKs, tools, and version claims, verify the applicable version against current
official primary documentation. Use discovery sources only to find the primary
source.

## Working approach

- Establish the requested outcome, affected Owner, material constraints, and
  completion criteria before editing.
- Inspect `git status`, relevant user changes, the target document, its Owner
  references, and nearby patterns. Preserve unrelated or overlapping user
  changes.
- Search narrowly with `rg` or `rg --files`. Do not load or rewrite the full
  Architecture corpus when a focused Owner chain is sufficient.
- Make the smallest coherent root-cause change. Avoid unrelated restructuring,
  terminology cleanup, bulk formatting, and speculative artifacts.
- Ask before adding a dependency, changing project-wide Architecture ownership,
  expanding the requested scope, or making a decision that has multiple
  materially different valid outcomes.
- Do not create implementation, Schema, Registry, Fixture, Receipt, generated
  output, or CI claims unless the task explicitly includes the corresponding
  artifact and verification.

## ChatGPT Pro collaboration route

- Apply this section only when the user explicitly invokes
  `$collaborating-with-chatgpt-pro` for work concerning this repository or its
  artifacts.
- Use the Codex in-app Browser and start each distinct outcome in a fresh standalone chat at `https://chatgpt.com/`.
- Browser ChatGPT is the MCP client that uses the verified Secure MCP
  Tunnel-backed `G Workspace Readonly` app; Codex only controls and verifies
  the Browser UI.
- Before sending, visibly verify that the chat is not Project-bound, the
  response-performance control is selected as `Pro`, the collapsed control
  reads `Pro`, and the exact `G Workspace Readonly` app is selected.
- Treat any path beginning with `/g/g-p-`, Project memory, Project Source,
  another app, API, Chrome, a lower response-performance option, or file
  attachment/upload/paste as forbidden fallbacks.
- This section selects the destination only. The Global Skill owns dynamic
  prompt generation, in-session transcript handling, adjudication, and stop
  rules. The repository policy below owns post-closure retention.
- Prefer the verified Secure MCP Tunnel-backed Filesystem MCP app for
  task-relevant artifacts under `G:\workspace`. Before relying on it, verify
  the allowed root, this repository's reachability, and the required tool
  capabilities. Do not infer read-only or write capability from the app name.
- If Tunnel access is unavailable or incomplete, stop as `blocked`. Do not
  attach, upload, paste, use Project Source, or fall back to another route.
  Tunnel reads and writes remain limited by the Task Contract, and
  ChatGPT-originated changes require local diff and validation by Codex.

## External consultation evidence retention

- This retention policy is a Miraikanai project decision for repository hygiene,
  not an OpenAI recommendation or an Architecture contract.
- Treat full consultation prompts, responses, screenshots, and attachment
  archives as transient working data. Keep them only until findings have been
  locally adjudicated, every accepted finding has been incorporated into its
  canonical Owner or ADR, the affected dependency／consumer closure has passed,
  and the requested terminal audit condition has been verified.
- Before deleting transient data, update `docs/reviews/README.md` with the audit
  ID and date, route and mode, scope, input document count and digest when
  available, valid-gap count, affected canonical Owners or ADRs, closure result,
  exact terminal response and marker, response digest when available, and
  retention disposition.
- After the summary is verified, delete per-round transcript Markdown and
  consultation attachment archives from the repository. Do not commit them as
  durable documentation. Read only the compact review summary by default; do
  not reopen consultation chats or reconstruct deleted transcripts unless the
  user explicitly requests it.
- Retain a full transcript only when the user explicitly requires retention or
  when an unresolved security, legal, licensing, incident, or external-audit
  obligation requires the original text. Record the reason and removal
  condition in the review summary.
- Review summaries are non-normative evidence. They never define current types,
  values, states, Gates, implementation, qualification, release, or Product
  completion; those claims remain with their canonical Owner and repository
  evidence.

## Architecture document rules

- Owner documents listed by the Architecture Index must retain the required
  header fields in the order defined by Architecture Governance. Header states
  must match repository evidence.
- Preserve Japanese as the current canonical Architecture prose language.
  Keep technical identities and canonical technical tokens in their defined
  English ASCII form. Do not translate user-authored source text.
- Keep one canonical Owner for each type, identifier, fixed value, Gate, state
  transition, algorithm, and diagnostic. Other documents link to the Owner and
  state usage conditions; they do not copy the full definition.
- Distinguish `official-spec`, `project-decision`, `provisional`, and `measured`.
  Do not describe a project choice as an external organization's official
  recommendation. Record external facts separately from Miraikanai adoption
  decisions.
- Do not infer implementation, activation, qualification, promotion, release,
  or Product completion from document existence or approval state.
- Keep `規範依存` acyclic and distinct from `関連文書`. Do not reverse the
  Product → Governance → Foundation → Authoring/Runtime → domain → Pack
  dependency direction, except for the Governance meta-contract described by
  the Owner.
- For new or moved documents, update the Owner header, canonical references,
  relative links, and typed anchors first. Update
  `docs/architecture/README.md` last.
- Create an ADR only for an architecturally significant choice with multiple
  credible alternatives. Do not rewrite the body of a `normative` or `rejected`
  ADR; create a successor and add reciprocal supersession metadata.
- Architecture Markdown filenames use lowercase kebab-case without a date.
  ADR filenames use `YYYY-MM-DD-<slug>.md`. Preserve repository LF line
  endings and the byte policy defined by `.gitattributes` and Naming/Project
  Layout.

## Verification

Run checks proportional to the change. At minimum:

```powershell
git diff --check
git status --short
git diff --stat
```

Also inspect the complete diff for every changed file and verify:

- required Owner header fields, state values, and evidence labels;
- unique canonical ownership and absence of stale duplicated definitions;
- relative Markdown links, typed fragments, anchors, and renamed references;
- Architecture Index entries when documents are added, moved, retired, or
  change status;
- external primary-source URLs, applicable versions, and
  `外部根拠確認日` when external claims change;
- unchanged bodies for `normative` and `rejected` ADRs;
- no statement upgrades a document-only change to implementation,
  qualification, activation, or completion.

No repository build, test runner, Markdown linter, link checker, Inventory
generator, or CI workflow is currently committed. Do not claim those checks
ran. If a required property cannot be verified manually, report it as a
remaining risk.

## Code Review Rules

Flag an Architecture change when it:

- omits or contradicts required header state;
- mixes an external fact with a project decision or calls the latter
  “official”;
- presents an absent or candidate artifact as current, materialized,
  generated, implemented, or verified;
- creates a second canonical definition instead of linking to the Owner;
- introduces a normative dependency cycle or an unresolved reference;
- changes immutable ADR body text;
- leaves the index, links, anchors, or supersession metadata inconsistent; or
- claims implementation, activation, qualification, release, or completion
  without repository evidence.

Prefer a concrete safe path in review feedback: identify the canonical Owner,
required evidence class, correct state wording, or successor ADR needed.
Formatting-only concerns belong in automated checks once the repository
provides them.

## Completion report

Lead with the outcome. Summarize changed files and affected Owner scope,
verification actually performed, checks that could not run, and remaining
risks or next actions. Never claim completion without current evidence.
