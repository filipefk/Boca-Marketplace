# Notas da release

Regras para redigir a `description` de uma release a partir do fluxo de mudanças (`notas:auto`).

## Modelo

```markdown
## Resumo

<2 a 4 frases, em linguagem de produto: o que esta versão entrega para quem usa. Sem jargão de commit.>

### Novidades
- <frase curta descrevendo a entrega> (#12, !34)

### Melhorias
- ...

### Correções
- ...

### Segurança
- ...

### Configuração de ambiente
- ...

### Documentação
- ...

### Outros
- ...

### Cards
- #12 <título do card>
- #15 <título do card>

### Merge requests
- !34 <título do MR> — @autor
- !36 <título do MR> — @autor

**Comparação:** [<PREV_TAG>...<TAG>](<web_url do projeto>/-/compare/<PREV_TAG>...<TAG>)
```

- Omita seções vazias.
- **Sem rodapé de autoria.**
- Cada item das seções por categoria descreve **uma entrega**, não um commit: agrupe o card, o(s) MR(s) e os commits que tratam da mesma coisa numa linha só, citando as referências entre parênteses.
- Use `#N` e `!N` puros — o GitLab transforma em link. **Nunca** escreva `Closes #N` / `Fixes #N` (em release não fecha nada, mas polui e confunde).
- Se o usuário pediu outro idioma ou outro formato (`notas:"em inglês"`, `notas:"só as correções"`), a instrução dele prevalece sobre este modelo.

## Categorização

Determine a categoria de cada entrega nesta ordem:

1. **Label de tipo do card** — as mesmas labels de tipo da skill `gitlab-card` ([card-rules.md](../../gitlab-card/references/card-rules.md)):

   | Label do card     | Seção nas notas            |
   |-------------------|----------------------------|
   | `Bug`             | Correções                  |
   | `Melhoria`        | Melhorias                  |
   | `Segurança`       | Segurança                  |
   | `Conf. Ambiente`  | Configuração de ambiente   |
   | `Documentação`    | Documentação               |
   | `Sanitizar`       | Outros                     |
   | `Tarefa`          | Outros (ou Novidades, se a tarefa entrega funcionalidade nova visível) |

   Comparação case-insensitive. Labels como `Feature`, `Story`, `Novidade` (se existirem no projeto) → **Novidades**.
2. **Labels do MR** — mesma tabela.
3. **Prefixo do título** do MR ou do commit (convenção de commits): `feat` → Novidades; `fix` → Correções; `perf`, `refactor` → Melhorias; `docs` → Documentação; `ci`, `build`, `chore` → Configuração de ambiente ou Outros; `security`, `sec` → Segurança.
4. Nenhum dos anteriores → julgue pelo conteúdo; na dúvida, **Outros**.

Card e MR com categorias diferentes → vale a do **card**.

## O que não entra

- Merge commits (`Merge branch ...`), commits de bump de versão, changelog, formatação, typos e reverts que se anulam dentro do mesmo intervalo.
- Commits sem MR entram em **Outros** só se forem relevantes para quem usa a versão.
- Cards ainda **abertos** entram, mas avise o usuário na confirmação — ele decide se mantém.
- MRs/issues de **outros projetos** só entram se o usuário pedir.

## Tamanho

Para releases grandes (muitos cards/MRs), mantenha as seções por categoria enxutas (agrupe itens parecidos) e deixe o detalhe para as listas de **Cards** e **Merge requests**. O limite da `description` é generoso (1.000.000 de caracteres), mas notas longas demais ninguém lê.
