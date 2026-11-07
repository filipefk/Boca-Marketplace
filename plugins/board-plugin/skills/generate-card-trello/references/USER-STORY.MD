# User story

Required template for the body of every card. It's the base on which the technical specifications ([API-SPEC.MD](API-SPEC.MD), [DATABASE-SPEC.MD](DATABASE-SPEC.MD), [INTEGRATION-SPEC.MD](INTEGRATION-SPEC.MD)) attach when applicable.

## Output format (follow exactly — output text itself must be in pt-BR, see "Output language" in `SKILL.md`)

```markdown
### **Descrição**

<resumo de 1-2 frases do objetivo da funcionalidade>

### **História de usuário**

- **Sendo** <perfil/papel>
- **Quero** <ação>
- **Para** <valor de negócio>

### **Critérios de aceite:**

#### Critério 01:

- **Dado** <contexto/precondição>
- **E** <precondição adicional, se houver>
- **Quando** <ação do usuário/gatilho>
- **Então** <resultado esperado>
- **E** <resultado adicional, se houver>

#### Critério 02:

...

### Especificação técnica:

<inserir aqui o(s) template(s) aplicável(is): API-SPEC.MD, DATABASE-SPEC.MD e/ou INTEGRATION-SPEC.MD>
```

## Filling rules

- **Description**: 1-2 sentences, based on the functional unit's introductory text in the source material.
- **As a/I want/So that**: the profile comes from permission tables or the text itself; if not explicit, infer from context but never leave blank. "So that" expresses the business value — infer if needed, never omit.
- **Acceptance criteria**: one testable functional requirement = one criterion. Number sequentially without skipping numbers. Don't combine two distinct behaviors in the same criterion. Include non-functional requirements that are testable via behavior (e.g., generic error message, access blocking). If rules differ by profile, create separate criteria per profile.
- **Technical specification**: never fill without first going through the knowledge verification chain in `SKILL.md` (source material → connectors → ask the user → never invent). Omit the section entirely if the unit has no applicable technical dimension.
