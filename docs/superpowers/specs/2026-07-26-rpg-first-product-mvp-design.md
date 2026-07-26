# RPG-First Product MVP Design

## 1. Status

- Date: 2026-07-26
- Scope: Product MVP、Reference Game、genre-neutral Core qualification、
  RPG Feature／Genre ownership boundary
- Decision: adopt a compact 2D command RPG as the first Product Reference
  Game; retain Shooter as a technical qualification consumer and later
  Reference Game
- Approval status: approved in product-direction review
- Implementation status: design only
- Implementation-plan status: intentionally absent
- Activation status: no Capability、Operation、Work Package、Product
  Definition, or Runtime Package is activated by this design

This design updates product direction only. It does not create C++、Shader、
Asset、test implementation, implementation tasks, calendar estimates, or an
implementation plan.

## 2. Decision

Miraikanai separates the first Product proof into two independent outcomes.

1. **Generic Engine C1** proves that Core、Editor、AI Authoring、Runtime、
   Save、Cook、Package, and Diagnostics work without a Genre Pack.
2. **RPG Reference MVP** proves that a beginner can use the same typed
   Authoring path to create, inspect, modify, package, and complete one compact
   data-rich game.

The RPG Reference MVP is the Product-facing First Playable. Shooter no longer
defines the Product MVP or Generic Core critical path. Shooter remains useful
for high-frequency Input、Collision、Physics、ECS、Rendering, and frame-budget
qualification, and may remain a later bundled Reference Game. Shooter evidence
cannot substitute for RPG Authoring evidence or genre-neutral Core evidence.

The first RPG is an original Miraikanai Reference Game. It does not reuse
Dragon Quest names、characters、monsters、story、maps、music、art、UI,
trademarks, or other protected creative expression.

## 3. Why RPG is the better Product MVP

| Product hypothesis | Compact RPG | Compact Shooter |
|---|---|---|
| Natural-language Game Brief to structured game | strong | medium |
| Data-driven rules and content authoring | strong | medium |
| AI and manual editing round-trip | strong | medium |
| Dialogue、UI、Localization、Accessibility | strong | weak／medium |
| Save、long-lived state、migration | strong | medium |
| Inventory、progression、cross-system references | strong | weak |
| High-frequency Input／Collision／Physics stress | weak／medium | strong |
| ECS／Rendering frame-budget stress | medium | strong |

Miraikanai's differentiator is the trusted, typed AI Authoring loop rather than
one specific Runtime mechanic. The Product MVP therefore prioritizes RPG
coverage and keeps the Shooter strengths in bounded technical fixtures. This
does not claim that RPG is the universally best engine benchmark.

## 4. Closed RPG Reference MVP boundary

### 4.1 Playable flow

The complete playable path is:

```text
Title／Continue／Settings
  -> Town
  -> Field
  -> Dungeon
  -> Boss battle
  -> Result／Ending
```

The fixture contains one World composition with one Town、one Field、one
Dungeon, and one Boss destination. Loading、Stage transition、Save／Load, and
Result must use their existing generic owners rather than RPG-private
substitutes.

### 4.2 Required gameplay coverage

The RPG Reference MVP requires:

- one player-controlled protagonist and one fixed companion;
- no recruitment or party reordering;
- one deterministic command battle flow with `Attack`、`Skill`、`Item`, and
  `Defend`;
- four regular enemy definitions and one Boss definition;
- at least one deterministic level-up that changes a declared gameplay value;
- at least one equipment change with a visible, typed gameplay effect;
- at least three Skill／spell definitions;
- at least one negative status effect with bounded duration and deterministic
  recovery;
- one mandatory Quest with one bounded choice that changes one Quest flag and
  subsequent dialogue but does not create a second ending;
- one buy-only Shop path that exercises currency、price、inventory capacity,
  acceptance, and rejection;
- one checkpoint plus manual Save／Load outside an active battle;
- `en-US` and `ja-JP` presentation with locale-invariant gameplay identity;
- keyboard and controller input on the Windows desktop Target;
- a complete manual Authoring run and a complete AI／manual round-trip run
  against the same semantic fixture.

Exact content counts above are Reference Game fixture bounds, not Generic Core
limits or public engine maximums.

### 4.3 Runtime cadence

The RPG MVP does not activate a global turn-based Simulation cadence. Runtime
continues to use the qualified fixed Simulation profile. Battle turns are an
RPG-owned deterministic state machine whose commands are accepted and applied
at existing Runtime boundaries.

This keeps Physics、Navigation、Animation、VFX、Input、Replay, and Debug
consumers on the current cadence contract. A future alternate global cadence
still requires its separately approved Product capability and full consumer
migration.

### 4.4 Explicit exclusions

The MVP excludes:

- job or class changes;
- recruitable or reorderable party members;
- branching endings;
- crafting;
- random loot tables;
- procedural dungeons;
- open-world streaming;
- complex economy simulation;
- unrestricted scripting;
- multiplayer;
- voice acting and cinematic cutscenes;
- commercial-quality bulk Asset generation;
- Android、Apple, or another Target as an MVP completion dependency.

These exclusions do not prohibit later Feature Packs. They prevent unreviewed
scope from entering the first Product proof.

## 5. Architecture split

The Product direction uses four layers.

| Layer | Responsibility |
|---|---|
| Generic Engine Core | Project State、typed ChangeSet、Runtime、World、UI、Input、Audio、Save、Cook、Package、Debug |
| Reusable RPG Feature Packs | command battle、actor progression、inventory／equipment、dialogue／quest、currency／shop |
| RPG Genre Pack | composition、RPG profile、game flow、command role mapping、Reference fixture binding |
| RPG Reference Game | original content、balance、World composition、localized presentation、acceptance fixture |

Existing Combat、Interaction、Scenario／Stage、Pickup／Grant、UI、
Localization、Save／Replay, and Runtime contracts are reused only where their
current semantics fit. RPG requirements do not silently broaden Shooter
contracts or place Genre semantics in Core.

This design deliberately does not assign final Stable IDs、Schema Fields、
owner document IDs, Work Package IDs, Capability IDs, or Operation IDs for the
new RPG surfaces. Each reusable domain requires a focused owner design before
Product Definition materialization. Placeholder IDs and inferred owners are
forbidden.

## 6. Data and authority flow

```text
Game Brief
  -> approved RPG Reference scope
  -> GameSpec
  -> typed RPG and generic Source documents
  -> ChangeSet
  -> preview／validate／approve／commit
  -> cooked Runtime artifacts
  -> play／save／load／replay
  -> package／install／offline completion
  -> sealed Evidence
```

RPG presentation never writes authoritative battle、Quest、inventory、
progression, or economy state. UI emits typed requests. The owning gameplay
System validates and publishes state through existing Runtime boundaries.

AI may explain and propose changes to bounded semantic projections. It cannot
invent missing RPG Capability activation, bypass owner validation, mutate
Runtime memory, or treat a successful Shooter fixture as RPG evidence.

## 7. Failure behavior

Every failure is source-preserving and typed.

- Missing RPG owner contract rejects Product Definition materialization.
- Missing or stale Feature Pack ref rejects Cook／Package.
- Invalid battle command preserves the last published battle state.
- Inventory／currency capacity failure performs no partial transaction.
- Quest／dialogue ref mismatch preserves the last valid Project revision.
- Save schema or content mismatch requires an approved migration or typed
  failure; it never loads a similar record.
- Locale mismatch cannot change Stable IDs、rules、balance, or Save identity.
- Missing Asset license／provenance blocks the affected package and does not
  substitute another Asset silently.
- Missing Target or performance evidence keeps the affected capability
  `not_activated`.

## 8. Qualification strategy

The RPG-first direction keeps three kinds of evidence separate.

1. **Genre-neutral Core holdout**
   - all Genre and optional Feature Packs uninstalled;
   - genreless World and Worldless UI／logic cases;
   - manual and AI Authoring、Save、Cook、Package, and Diagnostics.
2. **RPG Reference acceptance**
   - complete Title-to-Ending flow;
   - deterministic battle and Replay;
   - progression、equipment、Quest、choice、Shop, and Save round-trips;
   - paired `en-US`／`ja-JP` semantic invariance;
   - manual and AI changes converge to the same typed state.
3. **Technical stress fixtures**
   - Input／Collision／Physics action loop;
   - ECS allocation、query、structural mutation, and capacity;
   - Rendering frame、resource, and fallback budgets;
   - soak、failure atomicity, and clean-package checks.

No single fixture substitutes for another category. The RPG Reference Game is
not the maximum-scale performance benchmark, and the technical action fixture
is not the Product MVP.

## 9. Product Plan coordination

The Product Plan target direction changes as follows:

- C1 Product presentation and Pack portfolio use the compact 2D RPG;
- MVP-A becomes the RPG AI／manual Authoring First Playable;
- the Generic Core holdout remains an independent required gate;
- the 2D／3D Shooter remains a technical or later Reference consumer;
- Phase 6 may retain a compact 3D Shooter technical reference without making
  it the Product identity or a Generic Engine release dependency;
- the critical path ends in the RPG Reference MVP;
- scope reduction removes advanced and non-RPG Reference coverage before the
  RPG MVP contract.

Current Registry rows、Installed Product composition、active Operation set,
Capability activation rows, and Work Package lifecycle heads remain the
source baseline. This design does not edit those rows piecemeal.

RPG Product Definition materialization requires later, separately approved
owner designs and one atomic Product Definition Migration. Until then:

- current Shooter IDs and rows remain source history;
- no RPG Capability is reported as available;
- no current Shooter Operation is renamed to an RPG Operation;
- no Shooter Receipt is accepted as RPG qualification;
- no new RPG Work Package is scheduled.

## 10. Documents affected by this design

| Document | Design-level change |
|---|---|
| Product Plan | Record RPG-first MVP direction、scope、phase meaning、critical path, and source／destination separation |
| Pack Contract | Clarify that Reference Game choice is not a Core dependency and that current Shooter composition is not the future Product identity |
| Future RPG owner designs | Define reusable Feature owners、RPG Genre composition, Reference Game fixture, and atomic Product Definition destination |

Runtime、Rendering、Simulation、Platform, and current Pack owner documents are
not rewritten until a focused owner design finds a real contract change.

## 11. Acceptance criteria for the planning correction

The planning correction is complete when:

1. Product MVP names the compact 2D RPG as its first Product Reference Game.
2. Generic Core acceptance remains valid with every Genre Pack uninstalled.
3. Shooter is not a Generic Engine release prerequisite.
4. Shooter remains available as a technical qualification consumer without
   being presented as the Product MVP.
5. The RPG fixture boundary and exclusions are explicit.
6. Fixed Runtime cadence and RPG battle-state ownership are not conflated.
7. No unapproved RPG Stable ID、Schema、Capability、Operation, or Work Package
   is fabricated.
8. Current source Product Registry and Installed Product composition are not
   partially mutated.
9. No implementation task、estimate、source file, or implementation plan is
   created.
10. Existing user changes outside this design remain untouched.

## 12. Non-goals

- Implementing the RPG, engine, Editor, Runtime, or Pack.
- Writing an implementation plan or task breakdown.
- Scheduling Work Packages or estimating dates.
- Activating or migrating the current Product Definition.
- Replacing the Shooter Pack implementation or its historical records.
- Claiming general support for all RPGs or all game genres.
- Reproducing an existing commercial RPG or its protected content.
