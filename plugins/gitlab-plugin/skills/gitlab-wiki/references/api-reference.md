# Referência da API de Wikis (REST v4 do GitLab)

Instância de referência: **GitLab Community Edition v19.3.0**.
Documentação oficial: https://docs.gitlab.com/19.3/api/wikis/ (visão geral da API REST: https://docs.gitlab.com/19.3/api/rest/)

- Base: `${GITLAB_URL}/api/v4`
- Autenticação: header `PRIVATE-TOKEN: <token>` (alternativa: `Authorization: Bearer <token>`)
- Nenhum segredo vai em query string.

## Endpoints usados pela skill

| Operação | Método e caminho | Parâmetros relevantes |
|---|---|---|
| Buscar projetos do usuário | `GET /projects` | `membership=true`, `simple=true`, `search=<termo>`, `per_page=100` |
| Listar páginas da wiki | `GET /projects/:id/wikis` | `with_content=1` (opcional — traz o conteúdo de **todas** as páginas) |
| Ler uma página | `GET /projects/:id/wikis/:slug` | `render_html=true` (HTML renderizado), `version=<sha>` (versão específica) |
| Criar página | `POST /projects/:id/wikis` | `title` (obrigatório), `content` (obrigatório), `format` |
| Atualizar página | `PUT /projects/:id/wikis/:slug` | `content`, `title`, `format` (ao menos um) |
| Apagar página | `DELETE /projects/:id/wikis/:slug` | — |
| Anexar arquivo | `POST /projects/:id/wikis/attachments` | `file` (multipart, obrigatório), `branch` |

`format`: `markdown` (padrão), `rdoc`, `asciidoc`, `org`.

Wiki de grupo tem os mesmos seis endpoints em `/groups/:id/wikis`, mas é **Premium/Ultimate** — indisponível no CE.

## Formato das respostas

Listar (array) e ler/criar/atualizar (objeto) devolvem sempre os mesmos campos:

```json
{
  "content": "Aqui vai o conteúdo da página.",
  "format": "markdown",
  "slug": "deploy",
  "title": "deploy",
  "encoding": "UTF-8"
}
```

`GET /wikis` **sem** `with_content` omite o campo `content`. `DELETE` devolve `204` com corpo vazio.

Anexo devolve:

```json
{
  "file_name": "dk.png",
  "file_path": "uploads/6a061c4cf9f1c28cb22c384b4b8d4e3c/dk.png",
  "branch": "main",
  "link": {
    "url": "uploads/6a061c4cf9f1c28cb22c384b4b8d4e3c/dk.png",
    "markdown": "![dk](uploads/6a061c4cf9f1c28cb22c384b4b8d4e3c/dk.png)"
  }
}
```

Não existe id numérico de página nesses retornos: o identificador é o `slug`.

## Detalhes que importam

**Projeto por caminho** — `:id` aceita o id numérico ou o caminho URL-encoded. `grupo/projeto` vira `grupo%2Fprojeto`; subgrupos codificam cada barra (`grupo%2Fsub%2Fprojeto`).

**Slug no caminho** — o `:slug` também é URL-encoded: `Deploy/Rollback` vira `Deploy%2FRollback`. Se a barra não for codificada, o GitLab interpreta como outra rota e devolve `404`.

**`title` no `PUT` define o local da página** — o GitLab recalcula o caminho do arquivo a partir do `title` a cada `PUT`. Enviar um `title` diferente do caminho atual renomeia/move a página (e o GitLab grava um redirecionamento em `.gitlab/redirects.yml`). **Omitir o `title` também move**: numa subpágina, o `title` guardado é só a folha, então um `PUT` só com `content` joga `Pipelines/Pipeline-de-deploy` para a raiz como `Pipeline-de-deploy`. Para atualizar só o conteúdo sem mudar o local:

- página na raiz (slug sem `/`) → envie apenas `content`;
- subpágina (slug com `/`) → envie `content` **e** `title` com o slug atual completo (`Pipelines/Pipeline-de-deploy`).

Depois do `PUT`, confira que o `slug` devolvido é idêntico ao original. Ver [licoes-aprendidas.md](licoes-aprendidas.md#1).

**Escrita é sobrescrita total** — `PUT` com `content` substitui a página inteira; não há merge nem patch parcial. Leia antes, edite, envie o texto completo.

**Cada escrita é um commit** — a wiki é o repositório `<url-do-projeto>.wiki.git`. O histórico fica lá, então uma página apagada pode ser recuperada de um commit anterior. Não há lixeira na interface.

**Envio de conteúdo** — use `--data-urlencode "content@arquivo.md"` (form-urlencoded lendo do arquivo), nunca `-d` com o Markdown inline: quebras de linha, aspas e acentos quebram o escape do Git Bash no Windows. Se preferir JSON, o `content` teria que ir com `\n` escapado — mais frágil e sem ganho.

**Encoding** — a API trabalha em UTF-8 (`"encoding": "UTF-8"` na resposta). Arquivo local em ISO-8859-1 sobe com acentos corrompidos; converta antes.

**Paginação** — `GET /wikis` devolve todas as páginas de uma vez (não é paginado como issues). Ainda assim, `with_content=1` numa wiki grande gera uma resposta enorme — evite.

**Comentários de página** — existem endpoints (`/projects/:id/wiki_pages/:wiki_page_meta_id/notes`), mas eles usam um `wiki_page_meta_id` que **não** é o slug e **não** é devolvido por nenhum endpoint de wiki. Sem uma forma prática de obtê-lo pela REST, esta skill não implementa comentários. Se precisar, use a interface web ou a API GraphQL.

**Alternativa em massa** — para migrar dezenas de arquivos, exportar a wiki inteira ou reorganizar diretórios, clone o repositório da wiki e trabalhe com git:

```bash
git clone https://oauth2:${GITLAB_TOKEN}@gitlab.olfar.ind.br/grupo/projeto.wiki.git
```

Um push equivale a N chamadas da API, preserva mensagens de commit e evita o percent-encoding de slug. Cuidado: a URL acima contém o token — não a deixe registrada no `git remote` de um diretório versionado, e prefira `git credential` ou clonar por SSH.

## Códigos de erro comuns

| Código | Causa provável |
|---|---|
| `400` | Payload inválido — `title`/`content` ausente na criação, ou `format` desconhecido |
| `400` com corpo `Bad Request` **em texto puro** | Não é erro do GitLab: a requisição morreu no Rack, antes da aplicação. Quase sempre é acento passado inline na linha de comando (`--data-urlencode "title=…"`), corrompido pelo MSYS2 ao chamar o `curl` nativo. Passe o valor por arquivo (`title@arquivo`). Ver [licoes-aprendidas.md](licoes-aprendidas.md#2) |
| `401` | Token inválido, revogado ou expirado |
| `403` | Token sem escopo `api` (ou, em token fine-grained, sem a permissão de wiki), ou conta sem permissão de escrita na wiki do projeto |
| `404` | Página inexistente (slug errado ou barra não codificada), projeto inexistente, **wiki desabilitada** no projeto, ou token sem acesso (o GitLab devolve 404 em vez de 403 para não revelar o recurso) |
| `409` | Já existe página com esse slug — no `POST`, mude o título ou use `PUT` |

O corpo do erro vem como `{"message": "..."}` ou `{"error": "..."}`. Acrescente `-w "\n%{http_code}\n"` ao `curl` para ver o status junto com o corpo.

**Erro em JSON = o GitLab respondeu. Erro em texto puro = a requisição nem chegou nele.** Essa distinção economiza muito diagnóstico errado: um `Content-Type: text/plain` com `X-Runtime` de poucos milissegundos aponta para encoding ou payload malformado, não para regra de negócio.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina (o `python` do PATH é o stub da Microsoft Store e não executa). Para respostas grandes (listagem de páginas, conteúdo de página), redirecione a saída do `curl` para um arquivo no diretório de scratchpad da sessão (`-o`) e leia com a tool `Read`, em vez de tentar filtrar no shell. Para escrever conteúdo com acentos, use a tool `Write` — nunca `-d` com aspas escapadas.
