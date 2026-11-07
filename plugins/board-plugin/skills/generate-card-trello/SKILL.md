---
name: generate-card-trello
description: "Analyzes provided material (prompt, attached documents, connectors such as Drive/Gmail/wiki) and generates Trello cards with a user story, acceptance criteria in Given/When/Then, and technical specification (API, database, external integrations). Use whenever the user asks to turn requirements/specs/documents into Trello cards or tickets, break a spec or document into backlog items, or wants a full user story with acceptance criteria and technical specification drafted before the card is created. Not for a single quick card creation with no material to analyze — use the trello-card skill for that."
license: CC-BY-4.0
metadata:
  author: anderwll
  version: 1.0.1
---

# Generate Card Trello

Turns input material into Trello cards with traceable technical specification. Lean analysis. Zero invention. Confirmation before creation.

```
┌───────────┐   ┌─────────────┐   ┌────────────┐   ┌─────────────┐
│  ANALYZE  │ → │   DISCUSS   │ → │ STRUCTURE  │ → │ CREATE CARD │
└───────────┘   └─────────────┘   └────────────┘   └─────────────┘
   required        optional*         required          required

* Triggered only when gray areas exist — see "Discuss trigger"
```

## Critical rules (read before acting)

**Output language — pt-BR, always.** This skill's own documentation (this file and everything under `references/`) is written in English, but that is authoring language only. Every piece of **user-facing output must be written in Brazilian Portuguese (pt-BR)**, with no exception:
- All Discuss questions asked to the user (see [DISCUSS.MD](references/DISCUSS.MD)) — including when raised through the `AskUserQuestion` tool, when available.
- The full content of every card — title, description, user story, acceptance criteria, and technical specification — including the section labels themselves (e.g., "Descrição", "História de usuário", "Sendo/Quero/Para", "Critérios de aceite", "Dado/Quando/Então", "Rota", "Corpo", "Resposta", "Tabela", "Integração"). Use the pt-BR labels shown in the template output blocks in each reference file, not the English prose that surrounds them.
- The complementary `.md` file delivered at the end.

Only this `SKILL.md`/`references/` documentation itself stays in English — nothing produced for the user does.

**Loading this skill's files.** Reference files live under `references/`, inside this skill's own directory. Resolve paths relative to the skill directory — never the workspace root — and load them by name through the active skill. When reading a reference, **read it to the end** before acting; never act on a partial read.

**Execution contract — applies to every card, non-negotiable:**

1. Every functional unit from the input material becomes, at minimum, a user story with testable acceptance criteria — never a loose summary.
2. **Never invent** technical specification (route, body, response, table, integration payload). If the source material doesn't define those details, ask the user before filling the section. A placeholder is only acceptable if explicitly flagged as a pending assumption.
3. **Never create the card before the technical specification (when applicable) is confirmed by the user.** An incomplete card causes rework — present the full content for review first.
4. Card creation goes through **whatever Trello-capable tool/connector is available in the environment** (plugin skill, MCP, Zapier connector, etc.) — the exact mechanism is an implementation detail. When creating, **explicitly tell the user which mechanism was used** (e.g., "created via Zapier connector", "created via growdev-board-plugin:trello-card") so the origin stays traceable. See [GENERATE-CARD.MD](references/GENERATE-CARD.MD).
5. Besides the card, always also deliver a **complementary `.md` file** with the content of every story generated — a local record independent of Trello.

**Before Structure:** read [ANALYZE.MD](references/ANALYZE.MD) completely. If the analysis reveals gray areas, read [DISCUSS.MD](references/DISCUSS.MD) before assembling any technical specification.

## Discuss trigger

The **Discuss** phase is triggered from within Analyze when any of these signals appear:

- A field/rule looks ambiguous (e.g., required in one place, optional in another).
- The functional unit clearly involves an implicit technical dimension (persistence/data, external call, authentication, events) and the source material doesn't detail it.
- Access profiles or business rules diverge without the document explaining why.
- The user asked for a technical spec section (API, database, integration) but the provided material contains no route, schema, or payload to support it.

**It's safe to skip Discuss** when the provided material already contains all needed technical detail and there's no ambiguity in the acceptance criteria — in that case go straight to Structure.

## Output structure

```
[Trello card]              # story + confirmed technical specification (see TRELLO-CARD.MD)
[local .md file]           # same content, one `##` block per functional unit, complementary record
```

There is no persistent state between sessions for this skill — each analysis round is self-contained. Standard decisions (e.g., a 2xx/4xx/5xx response format already defined for an earlier card in the same conversation) should be reused within the same session instead of being re-asked.

## Workflow

1. **Analyze** — read the input material in full (prompt, attachment, connector link) and identify the functional units. See [ANALYZE.MD](references/ANALYZE.MD).
2. **Discuss** (if triggered) — raise the gray areas with the user before assembling the technical spec. See [DISCUSS.MD](references/DISCUSS.MD).
3. **Structure** — assemble the user story ([USER-STORY.MD](references/USER-STORY.MD)) and the applicable technical specification: [API-SPEC.MD](references/API-SPEC.MD), [DATABASE-SPEC.MD](references/DATABASE-SPEC.MD), [INTEGRATION-SPEC.MD](references/INTEGRATION-SPEC.MD) — more than one template can coexist in the same story.
4. **Present for review** — show the complete story (and, for large material, 1-2 examples first) before creating any card.
5. **Create card** — only after user confirmation. See [GENERATE-CARD.MD](references/GENERATE-CARD.MD).
6. **Deliver the complementary `.md`** file with all stories generated in the session.

## Commands

| Trigger pattern | Reference |
|---|---|
| Analyze document, identify units, break into backlog | [ANALYZE.MD](references/ANALYZE.MD) |
| Clarify gray area, ask a question, confirm a rule | [DISCUSS.MD](references/DISCUSS.MD) |
| Generate user story, acceptance criteria, Gherkin | [USER-STORY.MD](references/USER-STORY.MD) |
| Specify endpoint, route, API | [API-SPEC.MD](references/API-SPEC.MD) |
| Specify table, schema, database | [DATABASE-SPEC.MD](references/DATABASE-SPEC.MD) |
| Specify integration, external system, webhook | [INTEGRATION-SPEC.MD](references/INTEGRATION-SPEC.MD) |
| Create card, publish and send to the board | [GENERATE-CARD.MD](references/GENERATE-CARD.MD) |

## Knowledge verification chain

When filling in any technical specification section, follow this order strictly. Never skip steps.

```
Step 1: Source material provided (document, attachment, pasted text)
Step 2: Project connectors/context (Drive, Gmail, wiki, repository) — if available and relevant
Step 3: Ask the user directly (see DISCUSS.MD)
Step 4: Never invent — if there's no answer, leave the section as an explicit pending item, never as fact
```

**Rules:**

- Never jump to Step 4 if Steps 1-3 haven't been exhausted yet.
- Step 4 is never presented as fact — it's always flagged as a pending assumption.
- If the document has ambiguities (e.g., a field is required in one place and optional in another), flag it to the user instead of deciding silently.
