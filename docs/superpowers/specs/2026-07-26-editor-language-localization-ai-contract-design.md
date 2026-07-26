# Editor Language／Localization／AI Reply Contract Design

## 1. Status

- Date: 2026-07-26
- Scope: architecture contract coordination before implementation
- Decision: adopt English canonical technical vocabulary with localized presentation and independently configurable AI replies
- Implementation status: design only
- Migration status: not applicable because no released Editor preference schema, persisted Editor locale, or AI reply-locale contract exists

This design coordinates the existing Product、Naming、MCD、Editor、Project、
Localization、AI Verification owners. It does not create a new Architecture
owner, activate an MCD Operation, select an AI provider, or claim that a
multilingual Editor implementation exists.

## 2. Decision

Miraikanai Engine separates four language-bearing classes.

| Class | Canonical policy | Localization policy |
|---|---|---|
| Technical identity | English ASCII canonical vocabulary | Never localized |
| Machine-facing technical prose | Canonical `en-US` | Localized help is a separate projection |
| Product presentation | Localization key plus typed arguments | Resolved for the requested locale |
| User／Project-authored content | Preserve original UTF-8 and declared locale | Never translated implicitly |

Technical identity includes Stable IDs、MCD IDs、Schema／Field／enum names、
Operation and Diagnostic codes、module／namespace／target names、public API
identifiers、paths derived from Stable IDs, and generated code identifiers.

Machine-facing technical prose includes MCD `title`／`description`, Tool
descriptions, generated public API summaries, and canonical glossary terms.
English is used here for ecosystem interoperability and one stable vocabulary;
it is not treated as a security boundary or a guarantee of model correctness.

Product presentation includes Editor labels、menus、help、diagnostic messages,
accessible names, and AI natural-language replies. These strings are not
identity and do not enter semantic hashes, authorization subjects, ChangeSet
targets, or persistent references.

User／Project-authored content includes Project display names、Asset and Entity
display names、dialogue、Localization source messages、prompts、conversation
messages、comments written by the user, and other creative text. Original bytes
and declared locale remain authoritative. Translation may be proposed as a
separate change but is never an implicit rewrite.

## 3. Owner boundaries

| Concern | Canonical owner | Required consumers |
|---|---|---|
| English technical identifier grammar and canonical vocabulary | Naming／Project Layout | MCD、Editor、Project State、AI projections |
| MCD `title`／`description` canonical language | Executable Contracts | generated Tool／SDK projections |
| Editor display and AI reply preferences | Editor Workspace／UX | Editor UI、AI Partner |
| Locale-independent Editor semantic identity | Editor UI Framework | Workspace、UIA、AI、automation |
| Localization catalogs、BCP 47、fallback、Editor／Game catalog isolation | UI／Text／Localization／Accessibility | Editor、Project State、Cooker |
| Game／Project source locale and original user content | Project State plus Localization owner | Editor、AI、Cooker |
| Multilingual AI correctness and regression gates | AI Verification／Provenance | every Provider／Model／Prompt／Tool profile |
| Supported locale milestone and user-visible product promise | Product Plan | all subsystem owners |

No consumer copies another owner's schema, fallback algorithm, persistence
rule, or Eval threshold. It references the exact owner section instead.

## 4. Editor language preference

The Editor Workspace owner defines:

```text
EditorLanguagePreferencesV1
  schema_version = 1
  editor_display_locale: system | canonical BCP 47 locale
  ai_reply_locale: follow_editor | canonical BCP 47 locale
```

The record is per OS user and stored under the Editor User Profile outside the
Game Project. It is not exported with a Workspace, committed to Project State,
included in Undo, or packaged with a game.

C1 Editor display support and explicit AI reply-locale support are both the
exact set `en-US` and `ja-JP`. `system` resolves to a supported canonical
locale; an unsupported or unavailable system locale resolves to `en-US`.
Unsupported explicit values are rejected rather than silently approximated.
`en-US` is the Editor source locale and final fallback, but the first visible
language follows the system when supported. Additional locales require a
reviewed Product locale-set revision, qualified catalog resources, and
multilingual Eval coverage, not a preference-schema change.

Changing `editor_display_locale` re-resolves Editor presentation without
restarting the Editor. It may change layout and accessible display text, but
must not change Stable targets、typed values、available actions、Project
revision、semantic content hash, or an in-flight Task.

## 5. AI reply language

`follow_editor` is the default because it gives first-time users one coherent
language. An explicit BCP 47 locale is a separate persistent preference.

One conversation may hold an optional user-local reply-locale override. The
override affects new assistant prose only. It does not modify the persistent
preference, Project State, tool names, Schema fields, generated code
identifiers, or previous messages.

Each AI turn resolves and records:

```text
AiTurnLanguageContextV1
  input_language_tags[]
  requested_reply_locale
  effective_reply_locale
  preference_source: follow_editor | explicit_user_preference | conversation_override
```

This is request／trace metadata, not Project identity or authorization.
Changing Editor locale while a request is in flight does not retarget that
request. The next turn resolves the new preference. Conversation history keeps
its original text and per-message language metadata.

User input is sent in its original language. A provider-specific translation
may exist only as a disposable, attributable projection. It cannot replace the
original input, become a ChangeSet target, or bypass multilingual evaluation.

## 6. Localization catalog separation

Editor and Game localization use the shared Localization schema but separate
catalog artifacts and lifecycle.

| Catalog | Source locale | Required C1 display locale | Packaging |
|---|---|---|---|
| Editor catalog | `en-US` | `en-US`, `ja-JP` | Editor package only |
| Game Project catalog | Project-declared | Project-declared | Game package only |

The broader Engine conformance locale matrix remains a text-layout and
localization-engine qualification set. It is not a claim that every Editor
translation ships at C1.

Missing required Editor keys, argument-schema mismatch, or plural-branch
incompleteness fail the Editor language-pack build. Optional Editor text may
fall back to `en-US`, but the fallback emits a typed Localization diagnostic.
Editor and Game catalogs never fall back into each other.

## 7. Technical prose and generated source

- MCD `title` and `description` are canonical `en-US` technical prose.
- Localized help references MCD identity but is stored in a Localization
  catalog; it does not replace MCD text.
- Generated public API identifiers and generated-code identifiers use English
  canonical vocabulary.
- Generated public API summaries and generated doc comments default to
  canonical English.
- Existing user-authored comments and creative text are preserved.
- AI may explain generated code in the effective reply locale without changing
  the code or canonical API documentation.
- Architecture prose remains Japanese in the current repository. Exact
  technical tokens retain canonical spelling. A translated Architecture view
  is a non-authoritative projection unless Architecture Governance approves a
  separate migration.

## 8. Data flow

```text
Editor User Profile
  -> resolve effective Editor locale
  -> Editor Localization Catalog
  -> localized display／accessibility projection

Editor User Profile + optional conversation override
  -> resolve effective AI reply locale
  -> AiTurnLanguageContextV1
  -> original user text + typed semantic context + canonical technical schema
  -> structured Tool／ChangeSet result + localized assistant prose

Project LocalizationCatalogDocument
  -> Project-declared source and shipping locales
  -> Game Cooker／Runtime
```

The Editor locale does not feed Project source locale. The Game source locale
does not set the AI reply locale. Localized Editor labels never feed
authoritative AI target resolution.

## 9. Failure handling

| Failure | Required behavior |
|---|---|
| Invalid explicit locale tag | Reject preference change; retain last valid preference |
| Unsupported `system` result | Resolve to `en-US` and emit a non-blocking diagnostic |
| Missing required Editor translation | Fail language-pack build |
| Missing optional Editor translation | Fall back to `en-US` and emit diagnostic |
| Missing Game required translation | Apply existing Project package failure policy |
| Locale changes during AI request | Keep the request's resolved locale; apply change next turn |
| Provider returns wrong prose language | Mark Eval／turn language nonconformance; do not alter Tool result |
| Localized text changes semantic target or Tool arguments | Reject as a contract violation |
| Translation changes user-authored content implicitly | Reject ChangeSet |

## 10. Verification

The AI Verification owner adds a multilingual interaction suite with paired
`en-US` and `ja-JP` cases. Required cases include:

1. same intent in English and Japanese;
2. Japanese input with English canonical technical context;
3. mixed Japanese／English identifiers;
4. Editor locale switch between turns;
5. explicit AI reply-locale override;
6. unsupported system locale fallback;
7. user-authored Japanese names and dialogue;
8. missing Editor translation and pseudo-locale stress.

For paired mutation cases, Stable target、Tool selection、typed arguments、
ChangeSet primitive set, and semantic hash must be identical. Natural-language
wording is not required to be byte-identical; reply-locale adherence,
functional correctness, and evidence use are graded separately.

Hard gates are:

- unauthorized or invalid localized target acceptance: `0`;
- translated／invented technical identity in Tool output: `0`;
- implicit rewrite of user-authored content: `0`;
- required reply-locale mismatch: `0`;
- paired critical Tool／argument／ChangeSet invariant agreement: `100%`.

Every Provider、Model、Prompt、Tool Schema、context retrieval, or translation
projection update reruns this suite.

## 11. Coordinated document changes

| Document | Design change |
|---|---|
| Product Plan | Add language separation as a non-negotiable principle and C1 Editor locale promise |
| Naming／Project Layout | Define the four language classes and identifier／comment rules |
| Executable Contracts | Make MCD technical prose canonically `en-US` |
| Editor Workspace／UX | Own preference schema, resolution, persistence, switch behavior, and AI reply UX |
| Editor UI Framework | Guarantee semantic invariance across display locales |
| Project State | Keep Project source locale and user-authored content independent |
| UI／Text／Localization／Accessibility | Separate Editor and Game catalogs and define fallback／build gates |
| AI Verification／Provenance | Add multilingual paired Eval and release gates |
| Architecture Index | Add a routing entry without becoming a new owner |

## 12. Official external basis

The external sources justify separation of identity, source locale, and
localized presentation; they do not define Miraikanai schemas or defaults.

- Unity Editor language selection:
  <https://docs.unity.com/en-us/hub/add-editor-language>
- Unreal Engine localization targets and native culture:
  <https://dev.epicgames.com/documentation/en-us/unreal-engine/localization-overview-for-unreal-engine>
- Godot Editor language and raw／localized property names:
  <https://docs.godotengine.org/en/latest/classes/class_editorsettings.html>
- BCP 47 language tags:
  <https://www.rfc-editor.org/info/rfc5646/>
- ICU locale services:
  <https://unicode-org.github.io/icu/userguide/locale/>
- Separation of localizable and nonlocalizable resources:
  <https://learn.microsoft.com/en-us/globalization/internationalization/externalize-resources>
- OpenAI Structured Outputs:
  <https://developers.openai.com/api/docs/guides/structured-outputs>
- OpenAI evaluation guidance:
  <https://developers.openai.com/api/docs/guides/evaluation-best-practices>

## 13. Non-goals

- Translating Stable IDs、Schema fields、Tool names, or public API identifiers.
- Forcing all users to see English on first launch.
- Forcing a Game Project source locale to English.
- Rewriting existing Architecture prose into English.
- Treating English prompts as a substitute for typed validation or Eval.
- Claiming support for Editor locales beyond `en-US` and `ja-JP` at C1.
- Activating Provider routing, runtime translation, or an MCD Operation.
