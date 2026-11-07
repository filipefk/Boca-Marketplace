# Regras de revisão de merge requests

## Objetivo

Encontrar o que **precisa** mudar antes do merge e o que **vale a pena** melhorar — com evidência no código, não palpite. Uma revisão curta e certeira vale mais que uma longa e ruidosa.

## Ordem de leitura

1. **Descrição × alterações.** Leia a descrição do MR primeiro. Depois confira:
   - O código faz o que a descrição promete? Falta alguma parte anunciada?
   - Há alterações que a descrição não menciona (escopo extra, refatoração escondida, arquivo alterado por engano)?
   - A descrição está vazia ou não explica o porquê? Isso vira um comentário geral.
2. **Convenções do projeto** (`CLAUDE.md`, `CONTRIBUTING.md`, `.editorconfig`, quando existirem) — elas têm precedência sobre preferências genéricas.
3. **Arquivos**, na ordem: código de produção → migrações/scripts de dados → configuração/infra → testes → documentação.
4. **Threads existentes** — não repita o que já foi apontado; se um ponto antigo continua sem correção, responda na thread em vez de abrir outra.

## Checklist

| Aspecto | O que procurar |
|---|---|
| Correção | Lógica invertida, casos de borda (vazio, nulo, zero, lista com um item), off-by-one, comparações de tipo, fuso horário, arredondamento, estado compartilhado, concorrência |
| Tratamento de erro | Exceções engolidas, `catch` genérico sem log, erro que deixa dado pela metade, transação sem rollback, retorno de erro ignorado |
| Segurança | Injeção (SQL, comando, path), segredo ou credencial commitado, entrada não validada, autorização ausente, dado sensível em log, dependência nova de origem duvidosa |
| Dados e migrações | Migração irreversível ou sem rollback, lock longo em tabela grande, quebra de compatibilidade com a versão em produção, dado apagado sem backup |
| Testes | Mudança de comportamento sem teste, teste removido ou desabilitado, teste que não testa nada (sem assert), mocks que escondem o caso real |
| Performance | Consulta em loop (N+1), IO dentro de laço, carregar tabela/arquivo inteiro em memória, falta de índice para a nova consulta |
| Manutenção | Duplicação, nome enganoso, código morto, comentário desatualizado, `TODO` sem dono, complexidade desnecessária |
| Configuração | Variável de ambiente nova sem documentação, valor sensível em arquivo versionado, mudança de CI/Docker sem explicação |

## O que **não** comentar

- Formatação e estilo que o linter/formatter do projeto já cobre.
- Preferência pessoal sem ganho concreto ("eu faria com `map`").
- Código fora do diff — a menos que a mudança o quebre; nesse caso, comentário geral explicando a relação.
- Suspeitas sem evidência. Se não tem certeza, leia mais contexto (arquivo inteiro em `head_sha`); se continuar incerto, formule como pergunta, com severidade **Detalhe**.
- O mesmo problema repetido em vários lugares: **um** comentário no primeiro ponto, citando os demais (`mesmo padrão em src/b.py:10 e src/c.py:88`).

Como referência, uma revisão raramente precisa de mais de ~15 comentários. Se passar disso, agrupe.

## Severidade

| Severidade | Quando usar |
|---|---|
| **[Bloqueante]** | Bug, falha de segurança, perda de dado, quebra de build/teste, requisito da descrição não atendido. Deve ser corrigido antes do merge. |
| **[Sugestão]** | Melhoria concreta com ganho claro (legibilidade, performance, robustez), mas o MR funciona sem ela. |
| **[Detalhe]** | Ponto menor, opcional, ou pergunta para entender uma decisão. |

## Formato do comentário

````markdown
**[Sugestão]** <o problema, em uma frase>

<por quê: o impacto concreto — o que acontece, em que cenário>

```suggestion:-0+0
<linha corrigida, com a indentação original>
```

_Sugestão gerada pelo Claude Code_
````

- Uma ideia por comentário.
- Seja específico: nomeie a variável, a função, o cenário ("se `itens` vier vazio, `itens[0]` lança `IndexError`").
- Tom objetivo e respeitoso: fale do código, não da pessoa. Nada de "obviamente", "simplesmente".
- Use o bloco ```` ```suggestion ```` só quando a correção couber em poucas linhas e o comentário estiver ancorado numa linha da versão **nova** (`new_line`). O conteúdo substitui a(s) linha(s) inteira(s) — reproduza a indentação exata. `-0+0` = só a linha comentada; `-1+2` = uma acima e duas abaixo também.
- Escreva em português, a menos que a descrição e os comentários do MR estejam claramente em outro idioma — nesse caso, acompanhe o idioma do MR (o rodapé permanece o mesmo).
- **Todo** comentário publicado termina com o rodapé `_Sugestão gerada pelo Claude Code_`.

## Comentário geral (resumo)

Um único comentário geral fecha a revisão:

```markdown
**Resumo da revisão**

<1–3 frases: o que o MR faz e a impressão geral>

- <b> bloqueante(s), <s> sugestão(ões), <d> detalhe(s) — ver comentários nas linhas.
- <pontos que não cabem numa linha: descrição incompleta, falta de testes no geral, arquivos pulados>

_Sugestão gerada pelo Claude Code_
```

O resumo **não** aprova nem reprova o MR, e não usa palavras como "aprovado" ou "LGTM" — aprovação é decisão humana e fora do escopo da skill. "Sem bloqueantes encontrados" é aceitável.
