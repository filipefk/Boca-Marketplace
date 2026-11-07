# Referência da API REST v4 do GitLab — releases

Instância de referência: **GitLab Community Edition v19.3.0**.
Documentação oficial: https://docs.gitlab.com/19.3/api/rest/ — em especial
[Releases](https://docs.gitlab.com/19.3/api/releases/),
[Release links](https://docs.gitlab.com/19.3/api/releases/links/),
[Tags](https://docs.gitlab.com/19.3/api/tags/),
[Repositories (compare)](https://docs.gitlab.com/19.3/api/repositories/) e
[Commits](https://docs.gitlab.com/19.3/api/commits/).

- Base: `${GITLAB_URL}/api/v4`
- Autenticação: header `PRIVATE-TOKEN: <token>` (alternativa: `Authorization: Bearer <token>`)
- Nenhum segredo vai em query string.

## Endpoints usados pela skill

| Operação | Método e caminho | Parâmetros relevantes |
|---|---|---|
| Buscar projetos | `GET /projects` | `membership=true`, `simple=true`, `search`, `per_page=100` |
| Listar releases | `GET /projects/:id/releases` | `order_by=released_at\|created_at`, `sort=desc\|asc`, `per_page=100`, `include_html_description` |
| Ler release | `GET /projects/:id/releases/:tag_name` | `include_html_description=false` |
| Criar release | `POST /projects/:id/releases` | `tag_name` (obrigatório), `name`, `description`, `milestones[]`, `released_at` — **nunca `ref`** |
| Atualizar release | `PUT /projects/:id/releases/:tag_name` | `name`, `description`, `milestones[]`, `released_at` |
| Apagar release | `DELETE /projects/:id/releases/:tag_name` | — (a tag é mantida) |
| Listar links | `GET /projects/:id/releases/:tag_name/assets/links` | — |
| Criar link | `POST /projects/:id/releases/:tag_name/assets/links` | `name`, `url` (obrigatórios), `link_type`, `direct_asset_path` |
| Apagar link | `DELETE /projects/:id/releases/:tag_name/assets/links/:link_id` | — |
| Listar tags | `GET /projects/:id/repository/tags` | `order_by=version\|updated\|name`, `sort`, `search`, `per_page=100` |
| Ler tag | `GET /projects/:id/repository/tags/:tag_name` | — |
| Comparar | `GET /projects/:id/repository/compare` | `from`, `to` (tags, branches ou SHAs), `straight=false` |
| MRs de um commit | `GET /projects/:id/repository/commits/:sha/merge_requests` | — |
| Ler MR | `GET /projects/:id/merge_requests/:merge_request_iid` | — |
| Issues que o MR fecha | `GET /projects/:id/merge_requests/:merge_request_iid/closes_issues` | — |
| Issues relacionadas ao MR | `GET /projects/:id/merge_requests/:merge_request_iid/related_issues` | — |
| Ler issue | `GET /projects/:id/issues/:issue_iid` | — |
| Buscar milestone | `GET /projects/:id/milestones` | `title=<título exato>` |

## ⛔ Endpoints e campos proibidos

A skill **nunca** usa os itens abaixo, nem a pedido do usuário. Se pedirem, recuse e mostre os comandos `git` para ele rodar.

| Endpoint / campo | O que faria |
|---|---|
| `POST /projects/:id/repository/tags` | Cria tag |
| Campo `ref` no `POST /projects/:id/releases` | Cria a tag implicitamente a partir de um branch/SHA quando ela não existe |
| `DELETE /projects/:id/repository/tags/:tag_name` | Apaga tag |
| `POST/DELETE /projects/:id/protected_tags` | Mexe na proteção de tags |

## Detalhes que importam

**Tag no caminho** — `:tag_name` é URL-encoded: `/` → `%2F`, `.` → `%2E` (evita que `v1.4.0` seja interpretado como extensão/formato), espaço → `%20`. O mesmo vale para `from`/`to` em `compare`.

**Criar release sem tag** — sem `ref` e com tag inexistente, o GitLab devolve erro (`422`/`400`, mensagem do tipo `Ref is not specified`). A skill valida a tag antes (`GET .../repository/tags/:tag_name`) justamente para não chegar a esse ponto.

**`released_at`** — ISO 8601 (`2026-09-25T00:00:00Z`). No futuro → `upcoming_release: true`; no passado → `historical_release: true`. Não é possível remover um `released_at` já definido: só trocar por outra data.

**`milestones`** — array de **títulos**. No `PUT`, substitui a lista inteira; `[]` remove todos. Título inexistente → `400`.

**`description`** — Markdown; até 1.000.000 de caracteres. No `PUT`, substitui o texto inteiro.

**Resposta da release** — traz `tag_name`, `name`, `description`, `created_at`, `released_at`, `upcoming_release`, `author`, `commit` (`id`, `short_id`, `title`), `milestones[]`, `commit_path`, `tag_path`, `assets.count`, `assets.sources[]` (`format`, `url`), `assets.links[]` (`id`, `name`, `url`, `direct_asset_url`, `link_type`), `evidences[]` e `_links.self` (URL da release na interface).

**Links** — `link_type`: `other` (padrão), `runbook`, `image`, `package`. `direct_asset_path` (ex.: `/binaries/app.zip`) gera uma URL permanente `<_links.self>/downloads/binaries/app.zip`. A `url` precisa ser única dentro da release (duplicada → `400`/`422`). Apagar a release apaga os links.

**Compare** — `commits[]` traz os commits alcançáveis de `to` e não de `from` (com `straight=false`, a partir do merge-base). `diffs[]` pode ser truncado em comparações grandes; `compare_timeout: true` indica que o cálculo foi interrompido. `web_url` do compare: `<projeto>/-/compare/<from>...<to>`.

**Merge commit → MR** — a mensagem padrão do merge commit do GitLab termina com `See merge request <grupo/projeto>!<iid>`. Em projetos com **squash** ou **fast-forward** não há merge commit — por isso o fallback por `commits/:sha/merge_requests` (uma chamada por commit).

**`closes_issues` × `related_issues`** — o primeiro devolve as issues que o MR fecha no merge (`Closes #N` na descrição/commits); o segundo, as mencionadas. As duas listas podem se sobrepor.

**Ordem de tags** — `order_by=version` ordena por versionamento semântico (`v1.10.0` > `v1.9.0`), o que é o certo para achar a tag anterior. `updated` ordena pela data do commit; `name`, alfabética.

**Paginação** — o padrão é 20 itens por página. Envie sempre `per_page=100` (máximo). Os headers `X-Total`, `X-Total-Pages` e `X-Next-Page` indicam se há mais páginas; itere com `page=N`.

## Códigos de erro comuns

| Código | Causa provável |
|---|---|
| `400` | Payload inválido — milestone inexistente, `released_at` mal formatado, link com URL inválida/duplicada, ou texto com acento enviado pela linha de comando (corpo `Bad Request` em texto puro — ver lições aprendidas da wiki, §2) |
| `401` | Token inválido, revogado ou expirado |
| `403` | Token sem escopo `api` (ou `read_api` numa escrita), conta sem permissão (criar/editar/apagar release exige Developer), ou tag protegida sem permissão de criar tag protegida |
| `404` | Projeto, tag, release ou link inexistente — ou sem acesso (o GitLab devolve 404 em vez de 403 para não revelar o recurso) |
| `409` | Já existe release para essa tag |
| `422` | Dados rejeitados (ex.: tag inexistente sem `ref`, link duplicado) |

O corpo do erro vem como `{"message": "..."}` ou `{"error": "..."}`. Para ver o status junto com o corpo, acrescente `-w "\n%{http_code}"` ao `curl`.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Redirecione respostas grandes (compare, listagens) para um arquivo no diretório de scratchpad da sessão e leia com a tool `Read`. Para gerar JSON com acentos, use a tool `Write` — nunca `-d` com aspas escapadas, porque o Git Bash no Windows quebra o escape e corrompe acentos. Veja [lições aprendidas da wiki](../../gitlab-wiki/references/licoes-aprendidas.md), §2 e §6.
