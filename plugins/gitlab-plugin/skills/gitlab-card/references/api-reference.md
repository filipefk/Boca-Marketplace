# Referência da API REST v4 do GitLab

Instância de referência: **GitLab Community Edition v19.3.0**.
Documentação oficial: https://docs.gitlab.com/19.3/api/rest/

- Base: `${GITLAB_URL}/api/v4`
- Autenticação: header `PRIVATE-TOKEN: <token>` (alternativa: `Authorization: Bearer <token>`)
- Nenhum segredo vai em query string.

## Endpoints usados pela skill

| Operação | Método e caminho | Parâmetros relevantes |
|---|---|---|
| Buscar projetos do usuário | `GET /projects` | `membership=true`, `simple=true`, `search=<termo>`, `per_page=100` |
| Listar boards do projeto | `GET /projects/:id/boards` | — |
| Listar colunas do board | `GET /projects/:id/boards/:board_id/lists` | — |
| Listar labels do projeto | `GET /projects/:id/labels` | `per_page=100`, `search=<termo>` |
| Listar issues | `GET /projects/:id/issues` | `state=opened\|closed\|all`, `labels=<nomes>`, `search`, `milestone`, `assignee_username`, `order_by`, `sort`, `per_page=100` |
| Ler uma issue | `GET /projects/:id/issues/:issue_iid` | — |
| Criar issue | `POST /projects/:id/issues` | `title` (obrigatório), `description`, `labels`, `due_date`, `start_date`, `assignee_ids`, `milestone_id`, `confidential` |
| Atualizar issue | `PUT /projects/:id/issues/:issue_iid` | `title`, `description`, `add_labels`, `remove_labels`, `labels`, `due_date`, `start_date`, `state_event` |
| Listar comentários | `GET /projects/:id/issues/:issue_iid/notes` | `sort=asc`, `order_by=created_at`, `per_page=100` |
| Criar comentário | `POST /projects/:id/issues/:issue_iid/notes` | `body` (obrigatório) |

## Detalhes que importam

**`iid` × `id`** — a issue tem um `id` global (único na instância) e um `iid` interno (o número visível, `#42`). Os endpoints acima usam sempre o **`iid`**. Não confunda: o `id` retornado no JSON da issue não funciona nesses caminhos.

**Projeto por caminho** — `:id` aceita o id numérico ou o caminho URL-encoded. `grupo/projeto` vira `grupo%2Fprojeto`. Subgrupos codificam cada barra: `grupo%2Fsub%2Fprojeto`.

**Colunas do board** — `GET .../lists` retorna, para cada coluna, `id`, `position`, `max_issue_count` e um objeto `label` com `name`, `color` e `description`. As colunas **Open** e **Closed** existem no board mas **não** são retornadas. No CE as colunas só podem ser baseadas em label; listas por assignee, milestone e iteration são recursos Premium.

**Labels no payload** — `labels`, `add_labels` e `remove_labels` são **strings separadas por vírgula**, não arrays. `labels` substitui o conjunto inteiro (string vazia remove todas); `add_labels`/`remove_labels` fazem alteração incremental e são a forma correta de mover entre colunas. Labels que não existem são **criadas automaticamente** ao serem enviadas.

**Datas** — `due_date` e `start_date` (este desde a 19.1) usam `YYYY-MM-DD`. Campos de leitura como `created_at` e `updated_at` vêm em ISO 8601.

**Estado** — não existe arquivar. Use `state_event: "close"` e `state_event: "reopen"` no `PUT`. O campo de leitura é `state` (`opened` ou `closed`).

**Notes de sistema** — a lista de notes mistura comentários de pessoas com eventos automáticos. Os automáticos têm `"system": true` e devem ser descartados na exibição.

**Paginação** — o padrão é 20 itens por página. Envie sempre `per_page=100` (máximo). Os headers `X-Total` e `X-Next-Page` indicam se há mais páginas; se necessário, itere com `page=N`.

## Códigos de erro comuns

| Código | Causa provável |
|---|---|
| `400` | Payload inválido — campo obrigatório ausente ou valor mal formatado. Também: valor não numérico num parâmetro inteiro do caminho (ex.: `#42` no lugar de `42` em `:issue_iid`) — na 19.3 isso devolve `400`; em versões anteriores podia vir como `401` ou `404` |
| `401` | Token inválido, revogado ou expirado |
| `403` | Token sem escopo `api` (ou, em token fine-grained, sem a permissão do recurso), ou conta sem permissão para escrever no projeto |
| `404` | Projeto ou issue inexistente — ou o token não tem acesso ao projeto (o GitLab devolve 404 em vez de 403 para não revelar a existência do recurso) |
| `409` | Conflito ao atualizar |

O corpo do erro vem como `{"message": "..."}` ou `{"error": "..."}`. Para inspecionar o status junto com o corpo, acrescente `-w "\n%{http_code}"` ao `curl`.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Para respostas grandes (listagem de projetos, issues ou notes), redirecione a saída do `curl` para um arquivo no diretório de scratchpad da sessão e leia com a tool `Read`, em vez de tentar filtrar no shell. Para gerar JSON com acentos, use a tool `Write` ou heredoc em arquivo temporário — nunca `-d` com aspas escapadas, porque o Git Bash no Windows quebra o escape.
