# Modelos para criação de merge request

## Título (`title`)

Frase curta e objetiva (máximo 80 caracteres) resumindo o que o MR entrega, seguida obrigatoriamente do sufixo ` (Criado pelo Claude Code)` — o mesmo padrão dos títulos de issue da skill `gitlab-card`.

- Verbo no infinitivo, descrevendo o resultado: `Corrigir login com senha expirada (Criado pelo Claude Code)`.
- Não repita o nome do branch nem o número da issue no título.
- Com `draft:sim`, prefixe `Draft: ` — `Draft: Corrigir login com senha expirada (Criado pelo Claude Code)`. O GitLab trata esse prefixo como MR em rascunho (não pode ser mergeado até ser marcado como pronto).

Se o MR tiver um único commit com título bom, ele pode servir de base para o título do MR.

## Descrição (`description`)

O GitLab renderiza Markdown. Monte a descrição a partir dos commits (`compare.commits[]`) e das alterações (`compare.diffs[]`) — não copie as mensagens de commit cruas; reescreva de forma que um revisor entenda o todo.

```markdown
## Resumo

<1–3 frases: o que muda e por quê>

## Alterações

- <mudança 1, em linguagem de negócio ou técnica clara>
- <mudança 2>

## Como testar

1. <passo>
2. <passo>
3. <resultado esperado>

## Relacionado

- Relacionado a #<iid>
```

Regras:

- **Resumo** é obrigatório. Se não for possível inferir o porquê a partir dos commits e do código, pergunte ao usuário com `AskUserQuestion` antes de montar a descrição.
- **Alterações**: uma linha por mudança relevante; ignore commits de merge, `wip`, `fix typo` e afins (entram na mudança a que se referem).
- **Como testar**: passos concretos inferidos do código (endpoint, tela, comando, teste automatizado). Se não der para inferir, escreva `A definir pelo autor.` em vez de inventar.
- **Relacionado**: só inclua se houver issue identificada (pelo nome do branch ou informada pelo usuário) **e** confirmada via API. Omita a seção inteira caso contrário.
- **Nunca** use `Closes #N`, `Fixes #N` ou `Resolves #N` sem pedido explícito do usuário — essas palavras fecham a issue automaticamente quando o MR é mergeado. Use `Relacionado a #N`.
- Não inclua segredos, URLs internas de credenciais, dumps ou dados pessoais que apareçam no diff.
