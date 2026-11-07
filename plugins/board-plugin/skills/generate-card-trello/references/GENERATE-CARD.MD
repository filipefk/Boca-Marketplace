# Generate card in board

Last phase. Only run after the user confirms the complete story (description, user story, acceptance criteria, and technical specification, when applicable) — never before.

## Choosing the mechanism (generic)

This skill doesn't assume a single Trello integration path. When creating the card, use **whatever Trello-capable tool/connector is available in the environment**, in this order of preference:

1. **Dedicated skill or tool** (e.g., a plugin skill with "trello-card" capability), if available — it typically already resolves board/list by name and has a richer interface (checklists, labels, etc.).
2. **Generic connector plugged into the environment** (e.g., Zapier, or any MCP with a create card/ticket action for Trello).

If more than one is available, prefer the more specific option (1) over the generic one (2). If none is available, tell the user instead of simulating the creation.

**Always tell the user, after creating the card, which mechanism was used** (e.g., "card created via Zapier connector", "card created via the trello-card skill from the growdev-board-plugin plugin") — the origin needs to stay traceable, since different mechanisms may have different behaviors or limitations.

## Resolving board and list

- **List is fixed: always `BACKLOG`.** Every card created by this skill goes into the `BACKLOG` list/column — never ask the user which list to use, and never place a card elsewhere unless the user explicitly overrides this for a specific card.
- If the user already indicated the destination Trello board (in this session or a prior instruction), reuse it without asking again.
- If not indicated, ask for the board name before creating the first card of the session. Reuse that choice for subsequent cards in the same session, unless the user asks for a different destination.
- Resolve the board **by name**, never a hardcoded ID — confirm with the user if the search returns more than one match.

## Available labels

Apply every label that matches the card's content — a card can and often will carry more than one. Never invent a label outside this list; if none fit, leave the card unlabeled.

| Label | When to apply |
|---|---|
| `backend` | The unit involves backend/server-side work — API endpoints, business logic, background jobs. |
| `frontend` | The unit involves client-side/UI work — screens, components, client-side behavior. |
| `Release 1` | The unit is scoped for the Release 1 delivery. Apply when the user confirms this scope (don't assume by default). |
| `growauth-api` | The unit touches the growauth-api service (authentication/authorization). |
| `chatwoot` | The unit touches or integrates with Chatwoot. |
| `evolution-api` | The unit touches or integrates with Evolution API (WhatsApp integration). |

If the card content doesn't make the applicable label(s) obvious (e.g., unclear whether it's `backend` or `frontend`, or which system it touches), ask the user during Discuss rather than guessing (see [DISCUSS.MD](DISCUSS.MD)).

## Assembling the card

- **Title and description must be in pt-BR** — see "Output language" in `SKILL.md`. This applies to the actual card content, regardless of this file's own English prose.
- **Title**: the functional unit's name identified during Analyze, followed by ` (Created with Claude)` at the end (e.g., "Cadastro de usuário (Created with Claude)"). This suffix is always in English and always appended, regardless of the pt-BR rule for the rest of the title.
- **Description**: the complete markdown content of the story — Descrição, História de usuário, Critérios de aceite, Especificação técnica — exactly as presented and confirmed by the user, in pt-BR.
- **Labels**: apply the matching label(s) from "Available labels" above.
- One card per functional unit. Don't group multiple units into a single card.

## After creating

- Confirm the created card's link/identifier to the user, when the mechanism used returns that information.
- Also generate the complementary `.md` file (see `SKILL.md`, Workflow step 6): one block per functional unit, with `##` + unit name above each story block, as a local record independent of Trello.
