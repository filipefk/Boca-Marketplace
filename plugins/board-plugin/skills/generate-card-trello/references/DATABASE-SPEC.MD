# Technical specification — Database

Use when the functional unit involves creating or altering a persistence structure (SQL). Can coexist with [API-SPEC.MD](API-SPEC.MD) and/or [INTEGRATION-SPEC.MD](INTEGRATION-SPEC.MD) in the same story.

## Output format (follow exactly — output text itself must be in pt-BR, see "Output language" in `SKILL.md`)

```
Table: name_tables (
  column: TYPE|RULES,
  ...
)
```

## Filling rules

- **Table name and columns**: never invent. Comes from the source material, the schema already present in the project (check via connector/repository if available), or a direct answer from the user.
- **Types**: use the type convention of the database already in use in the project when known (e.g., `VARCHAR(255)`, `INT`, `TIMESTAMP`); if unknown, ask instead of assuming a SQL dialect.
- **Relevant constraints**: flag `NOT NULL`, `UNIQUE`, `FOREIGN KEY` and similar when defined in the material or confirmed by the user — don't omit constraints that change the testable behavior of an acceptance criterion (e.g., email uniqueness).
- If the unit involves more than one table, repeat the block once per table within the same technical specification section.
