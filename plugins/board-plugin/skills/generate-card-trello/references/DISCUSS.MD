# Discuss

Triggered from within Analyze when gray areas exist. Goal: close the gaps with the user before structuring any technical specification — never decide silently.

## When to ask

- **Ambiguous rule**: a field/behavior is described in conflicting ways across different parts of the material.
- **Undetailed technical dimension**: the functional unit clearly involves an API, database, or external integration, but the material doesn't specify the route/schema/payload.
- **Undefined access profile**: it's unclear who can perform the action.
- **Technical spec requested without a source**: the user wants the API/database/integration section filled in, but nothing in the material supports the content.

## How to ask

- **Ask in pt-BR.** Every question posed to the user in this phase must be written in Brazilian Portuguese, regardless of this file's own English prose — see "Output language" in `SKILL.md`.
- **Prefer the `AskUserQuestion` tool when it's available in the environment (e.g., Claude Code).** It structures the gray areas as selectable questions instead of free-text prose, which is faster for the user to resolve. When using it:
  - Write `question`, `header`, and every option's `label`/`description` in pt-BR — the language rule applies to this tool's fields exactly as it applies to plain text.
  - One question per independent gray area; group only the ones that share the same functional unit and genuinely belong together, up to the tool's limit (4 questions per call).
  - Always include a concrete recommended option when there's a reasonable default, marked "(Recomendado)" as the first option — but never let the presence of a recommendation replace flagging it as a pending assumption in the draft (see "Never fill a gap..." below).
  - If a gap doesn't fit a short set of discrete choices (e.g., asking for a free-form payload example, a full schema, or open-ended text), ask it as a plain pt-BR question instead of forcing it into `AskUserQuestion`.
  - If `AskUserQuestion` isn't available in the current environment, fall back to plain pt-BR questions in the conversation — the content and grouping rules below still apply either way.
- Group questions by functional unit — don't interrupt the flow question by question if several gaps belong to the same card.
- Be specific about what's missing: instead of "how does the API work?", ask "what's the HTTP method and route to create the registration? Which fields go in the body and which are required?" (in Portuguese, e.g., "qual o método HTTP e a rota para criar o cadastro? Quais campos vão no corpo e quais são obrigatórios?").
- If the default response format (2xx/4xx/5xx) was already defined for an earlier unit in the same session, **don't re-ask the whole pattern** — only ask what's different for this unit.
- Never fill a gap with a "reasonable" guess and move on. If the user hasn't answered yet, leave the section marked as an explicit pending item in the draft, never as fact.

## Output of this phase

For every functional unit that had gaps: the user's answers incorporated, ready to feed [USER-STORY.MD](USER-STORY.MD) and the applicable technical templates ([API-SPEC.MD](API-SPEC.MD), [DATABASE-SPEC.MD](DATABASE-SPEC.MD), [INTEGRATION-SPEC.MD](INTEGRATION-SPEC.MD)).

Units with no gaps skip this phase entirely — they go straight from Analyze to Structure.
