# Referência da API REST v4 do GitLab — merge requests

Instância de referência: **GitLab Community Edition v19.3.0**.
Documentação oficial: https://docs.gitlab.com/19.3/api/rest/ — em especial
[Merge requests](https://docs.gitlab.com/19.3/api/merge_requests/),
[Discussions](https://docs.gitlab.com/19.3/api/discussions/) e
[Draft notes](https://docs.gitlab.com/19.3/api/draft_notes/).

- Base: `${GITLAB_URL}/api/v4`
- Autenticação: header `PRIVATE-TOKEN: <token>` (alternativa: `Authorization: Bearer <token>`)
- Nenhum segredo vai em query string.

## Endpoints usados pela skill

| Operação | Método e caminho | Parâmetros relevantes |
|---|---|---|
| Usuário do token | `GET /user` | — |
| Buscar usuário | `GET /users` | `username=<usuario>` |
| Buscar projetos | `GET /projects` | `membership=true`, `simple=true`, `search`, `per_page=100` |
| Ler projeto | `GET /projects/:id` | — (`default_branch`) |
| Listar MRs | `GET /projects/:id/merge_requests` | `state=opened`, `author_username`, `reviewer_username`, `source_branch`, `target_branch`, `order_by`, `sort`, `per_page=100` |
| Ler MR | `GET /projects/:id/merge_requests/:merge_request_iid` | — |
| Diffs do MR | `GET /projects/:id/merge_requests/:merge_request_iid/diffs` | `page`, `per_page=100` |
| Arquivo numa revisão | `GET /projects/:id/repository/files/:file_path/raw` | `ref=<sha ou branch>` |
| Branch remoto | `GET /projects/:id/repository/branches/:branch` | — |
| Comparar branches | `GET /projects/:id/repository/compare` | `from=<destino>`, `to=<origem>` |
| Labels do projeto | `GET /projects/:id/labels` | `per_page=100` |
| Ler issue | `GET /projects/:id/issues/:issue_iid` | — |
| Listar threads | `GET /projects/:id/merge_requests/:merge_request_iid/discussions` | `page`, `per_page=100` |
| Nova thread (em linha) | `POST /projects/:id/merge_requests/:merge_request_iid/discussions` | `body`, `position` |
| Comentário geral | `POST /projects/:id/merge_requests/:merge_request_iid/notes` | `body` |
| Responder thread | `POST /projects/:id/merge_requests/:merge_request_iid/discussions/:discussion_id/notes` | `body` |
| Resolver/reabrir thread | `PUT /projects/:id/merge_requests/:merge_request_iid/discussions/:discussion_id` | `resolved=true\|false` |
| Listar rascunhos | `GET /projects/:id/merge_requests/:merge_request_iid/draft_notes` | — |
| Criar rascunho | `POST /projects/:id/merge_requests/:merge_request_iid/draft_notes` | `note`, `position`, `in_reply_to_discussion_id`, `resolve_discussion` |
| Publicar todos os rascunhos | `POST /projects/:id/merge_requests/:merge_request_iid/draft_notes/bulk_publish` | opcionais desde 19.3: `note`, `internal`, `reviewer_state` — a skill não os envia (ver "Draft notes") |
| Criar MR | `POST /projects/:id/merge_requests` | `source_branch`, `target_branch`, `title` (obrigatórios), `description`, `assignee_id`, `reviewer_ids`, `labels`, `remove_source_branch` |

## ⛔ Endpoints proibidos

A skill **nunca** chama os endpoints abaixo, nem a pedido do usuário. Se pedirem, recuse e devolva o `web_url` do MR.

| Endpoint | O que faria |
|---|---|
| `PUT /projects/:id/merge_requests/:iid/merge` | Merge |
| `GET /projects/:id/merge_requests/:iid/merge_ref` | Prepara ref de merge |
| `POST /projects/:id/merge_requests/:iid/cancel_merge_when_pipeline_succeeds` | Mexe no auto-merge |
| `PUT /projects/:id/merge_requests/:iid` (qualquer campo, inclusive `state_event=close`) | Fechar / "abandonar" / editar MR |
| `DELETE /projects/:id/merge_requests/:iid` | Apagar MR |
| `POST /projects/:id/merge_requests/:iid/approve` e `/unapprove` | Aprovar / desaprovar |
| `PUT /projects/:id/merge_requests/:iid/rebase` | Rebase |

No GitLab **não existe** estado "abandonado": abandonar um MR é fechá-lo sem merge. Por isso "abandonar" cai na mesma proibição de "fechar".

## Detalhes que importam

**`iid` × `id`** — o MR tem um `id` global e um `iid` interno (o número visível, `!42`). Os endpoints usam sempre o **`iid`**.

**Projeto, arquivo e branch no caminho** — `:id` aceita o id numérico ou o caminho URL-encoded (`grupo/projeto` → `grupo%2Fprojeto`). `:file_path` e `:branch` também são URL-encoded: `/` → `%2F`, e `.` → `%2E` (evita que a extensão seja interpretada como formato da resposta). O mesmo vale para `from`/`to` em `compare`.

**`diff_refs`** — o JSON do MR traz `diff_refs.base_sha` (merge base), `start_sha` (commit do branch de destino) e `head_sha` (último commit do branch de origem). Os três vão na `position` de todo comentário em linha. Se o MR receber push novo durante a revisão, os SHAs mudam — busque o MR de novo antes de criar os rascunhos se houver dúvida.

**Diffs** — `GET .../diffs` é paginado e devolve, por arquivo: `old_path`, `new_path`, `a_mode`, `b_mode`, `diff`, `new_file`, `renamed_file`, `deleted_file`, e conforme a versão `generated_file`, `collapsed` e `too_large`. Quando o diff é grande demais, o campo `diff` vem vazio — busque o arquivo inteiro com `repository/files/.../raw`. O endpoint antigo `GET .../changes` está **depreciado**: não use.

**`position` de comentário em linha** — campos:

| Campo | Obrigatório | Valor |
|---|---|---|
| `position_type` | sim | `text` |
| `base_sha`, `start_sha`, `head_sha` | sim | de `diff_refs` |
| `old_path`, `new_path` | sim | caminhos antes/depois (iguais se não houve renomeação) |
| `new_line` | conforme a linha | número na versão nova |
| `old_line` | conforme a linha | número na versão antiga |

- Linha **adicionada** → só `new_line`.
- Linha **removida** → só `old_line`.
- Linha de **contexto** (inalterada, visível no diff) → `new_line` **e** `old_line`.

A linha precisa estar dentro de um hunk do diff. SHAs, caminhos ou números errados geram `400` (mensagens como `line_code` inválido ou `Note {:line_code=>["can't be blank"...]}`) — ou, pior, um comentário que o GitLab aceita mas exibe fora do diff.

Em JSON, `position` vai como objeto aninhado:

```json
{"note": "...", "position": {"position_type": "text", "base_sha": "...", "start_sha": "...", "head_sha": "...", "old_path": "a.py", "new_path": "a.py", "new_line": 42}}
```

**Sugestões de código** — no corpo do comentário em linha, um bloco ```` ```suggestion:-0+0 ```` vira o botão "Apply suggestion" na interface. Só funciona em comentários ancorados na versão **nova** (`new_line`). `-N+M` estende a substituição N linhas acima e M abaixo da linha comentada.

**Draft notes** — disponível no tier Free (CE). Os rascunhos são **por usuário**: `GET .../draft_notes` devolve só os do dono do token, e ninguém mais os vê. `bulk_publish` publica **todos** os rascunhos do usuário naquele MR, inclusive os criados à mão pela interface — daí o aviso no fluxo de revisão. Publicar não aprova o MR. Resposta `204` sem corpo.

Desde a 19.3, o `bulk_publish` aceita três parâmetros opcionais: `note` (comentário de resumo publicado junto), `internal` (torna esse resumo interno) e `reviewer_state` (`reviewed` ou `requested_changes` — marca o estado de revisão do usuário no MR e dispara o webhook de revisão; a doc diz que **não** registra aprovação formal). A skill **não** envia nenhum deles: o resumo continua sendo um rascunho geral, e `reviewer_state` é um veredito de revisão — mesma lógica de "aprovar é um voto humano". Só use com pedido explícito do usuário.

**Notes de sistema** — as discussions misturam comentários de pessoas com eventos automáticos. Os automáticos têm `"system": true` e devem ser descartados na exibição.

**Threads resolvíveis** — só discussions com `resolvable: true` aceitam `resolved`. Comentários de revisão em linha são resolvíveis; muitos comentários gerais não.

**Criar MR** — `labels` é uma **string separada por vírgula**; labels inexistentes são **criadas automaticamente** (por isso a skill só envia as que já existem). `reviewer_ids` é um array de ids. Um título começando por `Draft:` cria o MR como rascunho. `remove_source_branch: true` marca o branch de origem para ser apagado quando alguém fizer o merge.

**Paginação** — o padrão é 20 itens por página. Envie sempre `per_page=100` (máximo). Os headers `X-Total`, `X-Total-Pages` e `X-Next-Page` indicam se há mais páginas; itere com `page=N`.

## Códigos de erro comuns

| Código | Causa provável |
|---|---|
| `400` | Payload inválido — campo obrigatório ausente, `position` incorreta, ou texto com acento enviado pela linha de comando (corpo `Bad Request` em texto puro — ver lições aprendidas da wiki, §2). Também: valor não numérico num parâmetro inteiro do caminho (ex.: `!42` no lugar de `42` em `:merge_request_iid`) — na 19.3 isso devolve `400`; em versões anteriores podia vir como `401` ou `404` |
| `401` | Token inválido, revogado ou expirado |
| `403` | Token sem escopo `api` (ou, em token fine-grained, sem a permissão do recurso), ou conta sem permissão (ex.: resolver thread sem ser Developer) |
| `404` | Projeto, MR, branch ou arquivo inexistente — ou sem acesso (o GitLab devolve 404 em vez de 403 para não revelar o recurso) |
| `405` | Operação não permitida no estado atual do MR |
| `409` | Já existe um MR aberto para o mesmo par de branches |
| `422` | Dados rejeitados (ex.: branch de origem inexistente, origem = destino) |

O corpo do erro vem como `{"message": "..."}` ou `{"error": "..."}`. Para ver o status junto com o corpo, acrescente `-w "\n%{http_code}"` ao `curl`.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Redirecione respostas grandes (listagens, diffs, discussions) para um arquivo no diretório de scratchpad da sessão e leia com a tool `Read`. Para gerar JSON com acentos, use a tool `Write` — nunca `-d` com aspas escapadas, porque o Git Bash no Windows quebra o escape e corrompe acentos. Veja [lições aprendidas da wiki](../../gitlab-wiki/references/licoes-aprendidas.md), §2 e §6.
