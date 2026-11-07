# Analyze

First phase, always required. Goal: fully understand the input material and identify the functional units that become cards, before writing a single line of story or specification.

## Accepted input sources

- **Direct prompt** — the user describes the feature in the text itself.
- **Attached document** — a file pasted or uploaded in the conversation (PRD, spec, meeting notes).
- **Connectors** — a Google Drive/Docs link, a Gmail thread, a wiki page (e.g., Azure DevOps Wiki), or any other source reachable via a tool available in the environment.

In all cases: **read the material in full before acting.** Don't skip sections, don't summarize prematurely, don't assume the beginning of the document represents the whole.

## Steps

1. **Load the material in full.** If it's a connector link, fetch the actual content (don't assume what it contains from the file/title name). If it's a long document, read it to the end before starting to identify units.

2. **Identify the functional units.** One unit = one card. Granularity rule:
   - One unit per coherent functional subsection (e.g., "User registration", "User deactivation"), following the material's own numbering/structure when it exists.
   - **Don't split by individual requirement** — too granular, adds noise to the board.
   - **Don't group an entire section** covering multiple distinct features into a single unit — too broad, the card becomes a disguised epic.
   - When the material has no clear structure (e.g., a free-form prompt), propose the split into units to the user before proceeding.

3. **List, per unit, what's already defined versus what's missing:**
   - Description/goal of the feature — usually extractable from the introductory text.
   - User profile/role and business value — infer from permission tables/context when not explicit, but never leave blank.
   - Testable rules (become acceptance criteria).
   - Technical dimensions present: API call, persistence/database, external integration. Mark each as "detailed in the material", "mentioned but undetailed", or "absent".

4. **Decide whether Discuss is needed.** See "Discuss trigger" in `SKILL.md`. If any technical dimension is "mentioned but undetailed" and the card requires that spec, or if there's ambiguity in the rules, go to [DISCUSS.MD](DISCUSS.MD) before structuring.

5. **If there are no gray areas**, go straight to structuring the story ([USER-STORY.MD](USER-STORY.MD)) and the applicable technical specifications.

## Output of this phase

A list of functional units, each with:
- Name (becomes the card title)
- One-sentence summary
- Applicable technical dimensions (API / database / integration / none)
- Status of each dimension: ready to structure, or pending clarification
