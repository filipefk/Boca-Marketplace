# Technical specification — External integration

Use when the functional unit depends on an external system (e.g., payment gateway, support system, third-party webhook). Can coexist with [API-SPEC.MD](API-SPEC.MD) and/or [DATABASE-SPEC.MD](DATABASE-SPEC.MD) in the same story.

## Output format (follow exactly — output text itself must be in pt-BR, see "Output language" in `SKILL.md`)


- **Integração:** <sistema externo, ex.: gateway de pagamento>
- **Evento/gatilho:** <o que dispara a integração>
- **Direção:** <saída (enviamos dados) | entrada (recebemos, ex.: webhook) | ambas>
- **Payload/response:** <payload ou campos relevantes>
- **Tratamento de falha:** <o que acontece se a integração falhar/timeout — retry, fila, notificação, bloqueio>


## Filling rules

- **External system and event/trigger**: never invent. Confirm with the source material or the user which system is involved and what triggers the call.
- **Data sent/received**: list only confirmed fields. If the full payload isn't known, list the fields that support the acceptance criteria and flag the rest as pending.
- **Failure handling**: if the material doesn't define the behavior on error/timeout, ask the user — this usually becomes its own acceptance criterion (e.g., "Given the integration fails, Then the system must X").
- If the unit involves more than one integration, repeat the block once per integration within the same technical specification section.
