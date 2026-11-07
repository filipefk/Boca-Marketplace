# Technical specification — API

Use when the functional unit involves an HTTP endpoint. Can coexist with [DATABASE-SPEC.MD](DATABASE-SPEC.MD) and/or [INTEGRATION-SPEC.MD](INTEGRATION-SPEC.MD) in the same story.

## Output format (follow exactly — output text itself must be in pt-BR, see "Output language" in `SKILL.md`)

**VERBO** `/endpoint-name`
**Authorization:** `Bearer USER_TOKEN`

**Body:**
```ts
{
  field: type,
  ...
}
```

**Response:**

```ts
{
  success: boolean,
  message: number,
  data: {
    field: type,
    ...
  },
}
```
**Errors:** 400, 404, 500 seguindo o padrão da aplicação.

---

## Filling rules

- **Method and route**: never invent. If the source material doesn't define them, ask the user (see [DISCUSS.MD](DISCUSS.MD)).
- **Body**: list only confirmed fields, with type. Mark which ones are required.
- **Response**: if the project already has an established 2xx/4xx/5xx response pattern from an earlier story in the same session, reuse it and only ask what's different (specific `data` fields, for example).
- If the unit involves more than one endpoint (e.g., create + list), repeat the `Route/Header/Body/Response` block once per endpoint, within the same technical specification section.
