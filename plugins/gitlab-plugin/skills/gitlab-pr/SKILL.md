---
name: gitlab-pr
description: Lista, lê, mostra as alterações, revisa e comenta merge requests (PRs) de um projeto do GitLab (Community Edition, API REST v4). Os comentários da revisão nascem como rascunho (review pendente) e só são publicados com confirmação. Também lê, responde e resolve threads, e cria MR a partir do branch local já enviado ao remoto. NÃO faz merge, NÃO fecha, NÃO abandona e NÃO aprova MRs. Usa as mesmas credenciais da skill gitlab-card — GITLAB_URL e GITLAB_TOKEN (e opcionalmente GITLAB_PROJECT).
---

## Como usar

```
/gitlab-pr listar [autor:"usuario"|autor:eu] [revisor:"usuario"|revisor:eu] [destino:"main"]
/gitlab-pr ler <!iid | iid | URL do MR>
/gitlab-pr alteracoes <ref> [arquivo:"src/app.py"]
/gitlab-pr revisar <ref> [foco:"segurança"]
/gitlab-pr publicar <ref>
/gitlab-pr comentar <ref> <texto>
/gitlab-pr comentar <ref> arquivo:"src/app.py" linha:42 <texto>
/gitlab-pr threads <ref> [abertas]
/gitlab-pr responder <ref> thread:<id> <texto>
/gitlab-pr resolver <ref> thread:<id>
/gitlab-pr reabrir-thread <ref> thread:<id>
/gitlab-pr criar [destino:"develop"] [revisor:"usuario"] [labels:"Melhoria"] [draft:sim]
```

"PR" e "MR" são a mesma coisa: o GitLab chama de *merge request*. Aceite os dois termos.

## ⛔ O que esta skill NÃO faz — nunca execute

Estas ações estão **fora do escopo** e **não devem ser executadas em nenhuma hipótese**, mesmo que o usuário peça explicitamente, mesmo que pareça conveniente no meio de outro fluxo:

| Ação | Por que é proibida | Endpoint que **nunca** deve ser chamado |
|---|---|---|
| **Fazer merge** do MR | Integra código no branch de destino — decisão humana | `PUT .../merge_requests/:iid/merge`, `.../merge_ref`, `.../cancel_merge_when_pipeline_succeeds` |
| **Fechar** o MR | Encerra o MR sem integrar | `PUT .../merge_requests/:iid` com `state_event=close` |
| **Abandonar** o MR | No GitLab não existe estado "abandonado": abandonar **é** fechar sem merge — mesma proibição | `PUT .../merge_requests/:iid` com `state_event=close`, `DELETE .../merge_requests/:iid` |
| Aprovar / desaprovar | Aprovação é um voto humano | `POST .../approve`, `POST .../unapprove` |
| Rebase, editar título/descrição/labels/assignees de MR existente | Fora do escopo desta versão | `PUT .../merge_requests/:iid`, `PUT .../rebase` |
| `git push` (qualquer forma) | A skill não altera o remoto por git | — |

**Regra simples para conferir:** esta skill **nunca** faz `PUT` nem `DELETE` em `/merge_requests/:iid` ou em subcaminhos dele. As **únicas** exceções são `PUT .../discussions/:discussion_id` (resolver/reabrir thread) e as chamadas de draft notes (`POST .../draft_notes`, `POST .../draft_notes/bulk_publish`).

Se o usuário pedir qualquer ação da tabela: **não chame a API**. Responda que a ação está fora do escopo da skill `gitlab-pr`, e devolva o `web_url` do MR para que ele a faça pela interface do GitLab.

Também **não** escreva `Closes #N` / `Fixes #N` na descrição de um MR novo sem pedido explícito — essas palavras fecham a issue automaticamente no merge. Use `Relacionado a #N`.

## Referências

- [Configuração do token](references/token-setup.md) — variáveis de ambiente e permissões (as mesmas da skill `gitlab-card`)
- [Referência da API v4](references/api-reference.md) — endpoints, `position` de comentário em linha, draft notes, paginação, erros e a lista de endpoints proibidos
- [Regras de revisão](references/review-rules.md) — o que avaliar, severidades, formato do comentário, sugestões, rodapé
- [Modelos de MR](references/mr-models.md) — título e descrição de um MR novo
- [Lições aprendidas da wiki](../gitlab-wiki/references/licoes-aprendidas.md) — §2 (acento em argumento de linha de comando → `400`) e §6 (dois shells) valem igualmente aqui

## Variáveis de ambiente

| Variável         | Obrigatória | Descrição                                                                    |
|------------------|-------------|------------------------------------------------------------------------------|
| `GITLAB_URL`     | sim         | URL base da instância, sem barra final (ex.: `https://gitlab.olfar.ind.br`) |
| `GITLAB_TOKEN`   | sim         | Personal Access Token com escopo `api`                                       |
| `GITLAB_PROJECT` | não         | Projeto padrão: caminho `grupo/projeto` ou id numérico                        |

São exatamente as mesmas variáveis usadas pelas skills `gitlab-card` e `gitlab-wiki`.

Valide no início de cada fluxo:

```bash
GITLAB_URL="${GITLAB_URL:?Variável GITLAB_URL não encontrada — veja references/token-setup.md}"
GITLAB_TOKEN="${GITLAB_TOKEN:?Variável GITLAB_TOKEN não encontrada — veja references/token-setup.md}"
API="${GITLAB_URL%/}/api/v4"
```

Toda chamada é autenticada pelo header `PRIVATE-TOKEN`. **Nunca imprima o valor de `GITLAB_TOKEN`** nem o inclua na URL — deixe sempre o shell expandir a variável dentro do `curl`. Todos os comandos desta skill são **Bash** (Git Bash); não misture sintaxe do PowerShell.

## Conceitos do GitLab

- O MR é identificado pelo **`iid`** (número visível, `!42`), não pelo `id` global. Os endpoints de projeto usam o `iid`.
- **`diff_refs`** (no JSON do MR) traz `base_sha`, `start_sha` e `head_sha` — os três são obrigatórios para comentar numa linha do diff.
- **Thread × note:** uma *discussion* (thread) agrupa *notes* (comentários). Comentários em linha sempre abrem uma thread. Notes com `"system": true` são eventos automáticos (push, mudança de label) e não comentários de pessoas.
- **Draft notes** são comentários em rascunho: só o autor os vê até publicar. `bulk_publish` publica **todos** os rascunhos do usuário naquele MR de uma vez — é o "Submit review" da interface. Publicar rascunhos **não** aprova o MR.
- Não existe estado "abandonado" — os estados são `opened`, `closed`, `merged` e `locked`.

## Instruções de execução

Ao ser invocada, verifique `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

0. Pede **merge, fechar, abandonar, aprovar, desaprovar, rebase ou editar** um MR → **recusa** (seção ⛔ acima). Não chame nenhuma API de escrita.
1. Começa com `listar` → **fluxo de listagem**.
2. Começa com `ler` (ou `ver`), ou contém uma URL `<GITLAB_URL>/<caminho>/-/merge_requests/<iid>` sem outro verbo → **fluxo de leitura**.
3. Começa com `alteracoes`, `alterações` ou `diff` → **fluxo de alterações**.
4. Começa com `revisar` (ou `review`) → **fluxo de revisão**.
5. Começa com `publicar` → **fluxo de publicação de rascunhos**.
6. Começa com `comentar` → **fluxo de comentário avulso**.
7. Começa com `threads` (ou `comentarios`) → **fluxo de threads**.
8. Começa com `responder` → **fluxo de resposta**.
9. Começa com `resolver` → **fluxo de resolução** (resolver).
10. Começa com `reabrir-thread` → **fluxo de resolução** (reabrir).
11. Começa com `criar` → **fluxo de criação de MR**.
12. Nenhum verbo reconhecido → pergunte com `AskUserQuestion` o que fazer, oferecendo as opções mais prováveis para o texto informado.

---

## Passo comum

### A. Referência do MR (`ref`) → `MR_IID`

Aceite:

- `123` ou `!123` → `MR_IID=123`
- URL `https://gitlab.olfar.ind.br/grupo/projeto/-/merge_requests/123` — com ou sem sufixos (`/diffs`, `/commits`, `#note_987`, `?...`) → `MR_IID` é o primeiro segmento numérico depois de `/-/merge_requests/`; o caminho do projeto é o trecho entre o host e `/-/merge_requests/`. Esse caminho **tem precedência** sobre `projeto:` e `GITLAB_PROJECT`.

Sem referência nos fluxos que exigem uma → execute o fluxo de listagem e pergunte com `AskUserQuestion` qual MR usar (máximo 4 opções).

### B. Projeto → `PROJECT_ID`

Ordem de precedência:

1. Caminho extraído da URL do MR, se houver.
2. Valor do parâmetro `projeto:"..."`.
3. Conteúdo de `$GITLAB_PROJECT`.
4. No fluxo `criar`: o remote `origin` do repositório local (ver o fluxo).
5. Nada informado → liste os projetos e pergunte com `AskUserQuestion` (máximo 4 opções; se houver mais, mostre os 4 primeiros e avise que há outros).

Se o valor contiver `/`, é um caminho: use-o direto como `:id`, **URL-encoded** (`grupo/projeto` → `grupo%2Fprojeto`; subgrupos codificam cada barra). Se for numérico, use-o direto. Caso contrário, busque pelo nome:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects?membership=true&simple=true&per_page=100&search=<nome>"
```

Case `name` ou `path_with_namespace` (case-insensitive). Zero resultados → erro e encerre. Mais de um → `AskUserQuestion`. Guarde `id` como `PROJECT_ID` e o `web_url`.

### C. Buscar o MR → `MR` e `DIFF_REFS`

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}" \
  -o "<scratchpad>/mr.json" -w "%{http_code}\n"
```

Leia com a tool `Read`. Guarde `diff_refs.base_sha`, `diff_refs.start_sha`, `diff_refs.head_sha`, `source_branch`, `target_branch`, `state`, `web_url`. Se `diff_refs` vier `null`, o GitLab ainda está preparando o diff (`detailed_merge_status: preparing`) — avise e peça para tentar em instantes.

### D. Enviar texto à API — sempre por arquivo

Todo payload com texto livre (comentário, resposta, título, descrição) vai num **arquivo JSON gravado com a tool `Write`** (UTF-8) em `<scratchpad>/`, enviado com `--data-binary @arquivo`:

```bash
curl -s -X POST "<endpoint>" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/payload.json" \
  -w "\n%{http_code}\n"
```

Nunca passe texto com acento pela linha de comando (`-d`, `--form "x=..."`): o Git Bash corrompe os bytes e o GitLab devolve `400` — ver [licoes-aprendidas.md §2](../gitlab-wiki/references/licoes-aprendidas.md#2). No JSON, escape `"`, `\` e quebras de linha (`\n`).

### E. Respostas grandes e paginação

Não há `jq` nem `python` no PATH. Salve respostas grandes (diffs, discussions, listagens) com `-o <scratchpad>/<nome>.json` e leia com a tool `Read`. Envie sempre `per_page=100`; se o header `X-Next-Page` vier preenchido (use `-D <scratchpad>/headers.txt` para inspecionar), repita com `page=N` até esgotar.

### F. Rodapé

Todo texto que a skill **publica** no MR (comentário de revisão, comentário avulso, resposta) termina com uma linha em branco seguida de:

```
_Sugestão gerada pelo Claude Code_
```

---

## Fluxo de listagem

### Passo 1 — Extrair parâmetros

De `$ARGUMENTS` (após `listar`): `projeto`, `autor`, `revisor`, `destino`. O valor `eu` em `autor`/`revisor` é o dono do token:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" "${API}/user" -o "<scratchpad>/me.json"
```

(use o campo `username`).

### Passo 2 — Resolver o projeto e listar

Execute a parte **B** do Passo comum.

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests?state=opened&order_by=updated_at&sort=desc&per_page=100&author_username=<autor>&reviewer_username=<revisor>&target_branch=<destino>" \
  -o "<scratchpad>/mrs.json"
```

Omita os filtros não informados.

### Passo 3 — Exibir

```
MRs abertos em <path_with_namespace> — <N>

- !<iid> [<title>](<web_url>)
  <author.username> · <source_branch> → <target_branch> · <Draft, se draft=true> · atualizado em <updated_at>
```

Lista vazia → informe que não há MRs abertos com esses filtros.

---

## Fluxo de leitura

### Passo 1 — Resolver e buscar

Execute as partes **A**, **B** e **C** do Passo comum.

### Passo 2 — Exibir

```
MR:          !<iid> — <title>
Projeto:     <path_with_namespace>
Estado:      <state><, Draft se draft=true>
Branches:    <source_branch> → <target_branch>
Autor:       <author.name> (@<author.username>)
Atribuído:   <assignees[].name ou "sem responsável">
Revisores:   <reviewers[].name ou "sem revisores">
Labels:      <labels ou "sem labels">
Merge:       <detailed_merge_status>
Pipeline:    <head_pipeline.status ou "sem pipeline">
Alterações:  <changes_count> arquivo(s)
URL:         <web_url>

Descrição:
<description, ou "(sem descrição)">
```

Ao final, ofereça `alteracoes` e `revisar` para o mesmo MR.

---

## Fluxo de alterações

### Passo 1 — Resolver e buscar os diffs

Execute as partes **A**, **B** e **C** do Passo comum.

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/diffs?per_page=100&page=1" \
  -D "<scratchpad>/diffs-headers.txt" \
  -o "<scratchpad>/diffs-1.json"
```

Pagine conforme a parte **E**. **Não use** `GET .../changes` — está depreciado.

Cada item traz `old_path`, `new_path`, `new_file`, `renamed_file`, `deleted_file`, `diff` (o texto do diff do arquivo, começando pelo `@@`) e, conforme o caso, `too_large`, `collapsed` e `generated_file`.

### Passo 2 — Exibir

Sem `arquivo:` → resumo por arquivo, contando as linhas do `diff` que começam com `+` e `-` (ignorando os cabeçalhos `@@`):

```
!<iid> — <title> · <N> arquivo(s)

  M  src/app.py                (+12 −3)
  A  src/novo.py               (+40 −0)
  D  src/velho.py              (+0 −25)
  R  docs/a.md → docs/b.md     (+1 −1)
  ⚠  package-lock.json         (diff grande — omitido pelo GitLab)
```

Com `arquivo:"..."` → exiba o `diff` daquele arquivo em bloco ```` ```diff ````.

Arquivo com `too_large` ou `collapsed` (diff vazio) → para ver o conteúdo, baixe o arquivo inteiro na versão do MR. Codifique o caminho (`/` → `%2F`, `.` → `%2E`):

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/repository/files/<caminho_enc>/raw?ref=<head_sha>" \
  -o "<scratchpad>/head-<nome>"
```

Para a versão anterior use `ref=<base_sha>` e o `old_path`.

---

## Fluxo de revisão

Avalia o MR e cria os comentários como **rascunho** (draft notes). Nada fica visível para outras pessoas até o usuário confirmar a publicação.

### Passo 1 — Coletar o contexto

1. Partes **A**, **B** e **C** do Passo comum. Se `state` for `merged` ou `closed`, avise e pergunte com `AskUserQuestion` se quer revisar mesmo assim.
2. Diffs de todos os arquivos (Passo 1 do fluxo de alterações).
3. Threads existentes (Passo 1 do fluxo de threads) — para **não repetir** o que já foi apontado e para saber o que já foi resolvido.
4. Rascunhos já existentes do usuário neste MR:

   ```bash
   curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/draft_notes" \
     -o "<scratchpad>/drafts-existentes.json"
   ```

5. Convenções do projeto, se existirem no branch de destino: `CLAUDE.md`, `CONTRIBUTING.md`, `.editorconfig` (endpoint `repository/files/.../raw?ref=<target_branch>`; `404` = não existe, siga em frente).
6. Quando o trecho do diff não bastar para julgar (função cortada, variável definida fora do hunk), baixe o arquivo inteiro em `head_sha` e leia com `Read`.

Se `foco:"..."` foi informado, priorize esse aspecto, sem ignorar problemas graves de outra natureza.

### Passo 2 — Avaliar

Siga [review-rules.md](references/review-rules.md): primeiro confronte a **descrição** com as **alterações** (o MR faz o que diz? faz algo que não diz?), depois percorra os arquivos com o checklist. Pule arquivos gerados (`generated_file`, lockfiles, binários, minificados) e avise quais pulou.

### Passo 3 — Montar e exibir os achados

Para cada achado, determine a **posição** conforme a seção "Posição de um comentário em linha" abaixo. Exiba no chat:

```
Revisão de !<iid> — <title>
<N> achado(s): <b> bloqueante(s), <s> sugestão(ões), <d> detalhe(s)

1. [Bloqueante] src/app.py:42 (linha nova)
   <texto do comentário>

2. [Sugestão] src/db.py:17 (linha removida)
   <texto do comentário>

Comentário geral (resumo):
<texto>
```

Sem nenhum achado relevante → diga isso, **não** crie rascunhos, e encerre (ofereça publicar só o comentário de resumo, se o usuário quiser).

### Passo 4 — Confirmar

Use `AskUserQuestion` com as opções **Criar todos como rascunho**, **Escolher quais** e **Cancelar**.

- **Escolher quais** → peça os números (ex.: `1, 3, 4`) e se o comentário geral entra.
- **Cancelar** → encerre sem chamar a API.

Se `drafts-existentes.json` não estiver vazio, avise **antes** de criar: "Você já tem <N> rascunho(s) neste MR. Ao publicar, eles serão publicados junto."

### Passo 5 — Criar os rascunhos

Para cada item confirmado, grave um JSON com a tool `Write` (`<scratchpad>/draft-01.json`, `draft-02.json`, ...) e envie:

**Comentário em linha:**

```json
{
  "note": "**[Sugestão]** ...\n\n_Sugestão gerada pelo Claude Code_",
  "position": {
    "position_type": "text",
    "base_sha": "<diff_refs.base_sha>",
    "start_sha": "<diff_refs.start_sha>",
    "head_sha": "<diff_refs.head_sha>",
    "old_path": "<old_path>",
    "new_path": "<new_path>",
    "new_line": 42
  }
}
```

**Comentário geral** (resumo, ou achado sem linha no diff): só `{"note": "..."}`.

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/draft_notes" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/draft-01.json" \
  -w "\n%{http_code}\n"
```

Sucesso é `201`. Se um comentário em linha devolver `400` por causa da posição, **não desista do achado**: recrie-o como rascunho geral, começando por `` `arquivo:linha` — `` no texto, e informe no relatório.

### Passo 6 — Publicar ou deixar em rascunho

Mostre quantos rascunhos foram criados (e quais caíram como comentário geral) e o `web_url` do MR. Use `AskUserQuestion`:

- **Publicar agora** → fluxo de publicação, Passo 3 (sem nova confirmação — esta já é a confirmação).
- **Deixar em rascunho** → informe que os rascunhos estão visíveis só para o usuário, que podem ser editados ou apagados na interface do GitLab, e que a publicação é feita pelo botão **Submit review** ou por `/gitlab-pr publicar <ref>`.

### Posição de um comentário em linha

O `diff` de cada arquivo é dividido em hunks com cabeçalho `@@ -<ini_antigo>[,<qtd>] +<ini_novo>[,<qtd>] @@`. Percorra as linhas do hunk com dois contadores (`antigo = ini_antigo`, `novo = ini_novo`):

| Linha começa com | Tipo | Campos na `position` | Depois |
|---|---|---|---|
| `+` | adicionada | só `new_line = novo` | `novo++` |
| `-` | removida | só `old_line = antigo` | `antigo++` |
| ` ` (espaço) | contexto | `old_line = antigo` **e** `new_line = novo` | `antigo++`, `novo++` |
| `\ No newline at end of file` | — | ignore | — |

Regras:

- Só é possível comentar em linhas que **aparecem no diff** (dentro de algum hunk). Linha fora do diff → comentário geral citando `arquivo:linha`.
- **Confira o número antes de enviar:** para `new_line`, baixe o arquivo em `head_sha` e confirme com o `Read` (que mostra números de linha) que o conteúdo daquela linha é o esperado; para `old_line`, o mesmo com o arquivo em `base_sha`. Contar hunks de cabeça erra com facilidade.
- Prefira ancorar o comentário numa linha **adicionada**.
- Arquivo renomeado: `old_path` e `new_path` diferentes; arquivo novo ou apagado: os dois iguais ao único caminho.

---

## Fluxo de publicação de rascunhos

### Passo 1 — Resolver e listar os rascunhos

Partes **A** e **B** do Passo comum, depois:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/draft_notes" \
  -o "<scratchpad>/drafts.json"
```

Nenhum rascunho → informe e encerre.

### Passo 2 — Confirmar

Exiba cada rascunho (`arquivo:linha` a partir de `position`, quando houver, e o `note`). Use `AskUserQuestion`: **Publicar todos** ou **Cancelar**. Deixe claro que a publicação notifica o autor e os revisores do MR.

### Passo 3 — Publicar

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/draft_notes/bulk_publish" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -w "\n%{http_code}\n"
```

Sucesso é `204`. Reporte quantos comentários foram publicados e o `web_url` do MR. Publicar **não** aprova o MR.

---

## Fluxo de comentário avulso

Publica **direto** (sem rascunho) um único comentário pedido pelo usuário.

### Passo 1 — Extrair parâmetros

De `$ARGUMENTS` (após `comentar`): a referência, `arquivo:"..."` e `linha:N` (opcionais) e o texto. Sem texto → peça com `AskUserQuestion`. Execute as partes **A**, **B** e **C** do Passo comum.

Com `arquivo` e `linha`: busque os diffs e determine a posição ("Posição de um comentário em linha"), considerando `linha` como número na versão **nova** do arquivo. Se a linha não estiver no diff, avise e ofereça publicar como comentário geral citando `arquivo:linha`.

### Passo 2 — Confirmar

Exiba o MR, a posição (ou "comentário geral") e o texto com o rodapé. `AskUserQuestion`: **Comentar** ou **Cancelar**.

### Passo 3 — Publicar

**Geral** — JSON `{"body": "..."}`:

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/notes" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/comentario.json" \
  -w "\n%{http_code}\n"
```

**Em linha** — JSON `{"body": "...", "position": { ... }}` (mesmos campos de `position` do fluxo de revisão), enviado para `.../merge_requests/${MR_IID}/discussions`.

Reporte o sucesso (`201`) com o `web_url` do MR.

---

## Fluxo de threads

### Passo 1 — Buscar as discussions

Partes **A** e **B** do Passo comum, depois:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/discussions?per_page=100&page=1" \
  -D "<scratchpad>/disc-headers.txt" \
  -o "<scratchpad>/discussions-1.json"
```

Pagine conforme a parte **E**. **Descarte** as discussions cujas notes são todas `"system": true`.

### Passo 2 — Exibir

Com `abertas`, mostre só as threads com `resolvable: true` e `resolved: false`.

```
Threads de !<iid> — <N> (<a> abertas, <r> resolvidas)

[<8 primeiros caracteres do id>] <✔ resolvida | ○ aberta | · não resolvível>  <new_path:new_line, se houver position>
  <author.name> em <created_at>:
  <body>
  ↳ <author.name> em <created_at>:
    <body das respostas>
```

O id curto é o que o usuário usa em `thread:<id>`. Ao receber um id curto, case pelo prefixo com a lista completa; mais de um candidato → `AskUserQuestion`.

---

## Fluxo de resposta

### Passo 1 — Resolver

De `$ARGUMENTS` (após `responder`): referência, `thread:<id>` e texto. Partes **A** e **B** do Passo comum; resolva o `discussion_id` completo pelo fluxo de threads. Sem `thread:` → mostre as threads abertas e pergunte com `AskUserQuestion`. Sem texto → peça.

### Passo 2 — Confirmar

Exiba a thread (primeira note e últimas respostas) e a resposta com o rodapé. `AskUserQuestion`: **Responder** ou **Cancelar**.

### Passo 3 — Publicar

JSON `{"body": "..."}` gravado com `Write`:

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/discussions/<discussion_id>/notes" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/resposta.json" \
  -w "\n%{http_code}\n"
```

Reporte o sucesso (`201`) com o `web_url` do MR.

---

## Fluxo de resolução

Serve para `resolver` (`resolved=true`) e `reabrir-thread` (`resolved=false`).

### Passo 1 — Resolver e validar

Partes **A** e **B** do Passo comum; resolva o `discussion_id` pelo fluxo de threads. A thread precisa ter `resolvable: true` — senão, informe que ela não pode ser resolvida (comentários gerais que não são revisão, por exemplo) e encerre. Se já estiver no estado pedido, informe e encerre.

### Passo 2 — Confirmar

Exiba a thread. Ao **resolver**, avise que em projetos que exigem "todas as threads resolvidas" isso pode liberar o merge do MR. `AskUserQuestion`: **Resolver thread** / **Reabrir thread** (conforme o caso) ou **Cancelar**.

### Passo 3 — Atualizar

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/merge_requests/${MR_IID}/discussions/<discussion_id>" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --data "resolved=true" \
  -w "\n%{http_code}\n"
```

(`resolved=false` para reabrir.) Reporte o novo estado e o `web_url`.

---

## Fluxo de criação de MR

Cria um MR a partir do **branch atual do repositório local** (diretório de trabalho). A skill **não faz push**: o branch precisa já estar no remoto.

### Passo 1 — Validar o repositório local

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --porcelain
```

- Fora de um repositório git → informe e encerre.
- `branch --show-current` vazio → HEAD destacado (detached): informe e encerre.
- `status --porcelain` com saída → avise que há mudanças não commitadas que **não** entrarão no MR e pergunte com `AskUserQuestion` se segue (**Seguir assim** / **Cancelar**).

Guarde `SOURCE_BRANCH` e `LOCAL_SHA=$(git rev-parse HEAD)`.

### Passo 2 — Resolver o projeto pelo remote

```bash
git remote get-url origin | sed -E 's#//[^@/]+@#//#'
```

O `sed` remove credenciais embutidas (`https://oauth2:<token>@host/...`) — **nunca** exiba a URL do remote sem ele. Formatos aceitos:

- `https://host/grupo/projeto.git`
- `git@host:grupo/projeto.git`
- `ssh://git@host[:porta]/grupo/projeto.git`

Extraia o host e o caminho (sem `.git`). Se o host não for o de `GITLAB_URL`, avise e encerre. Sem `origin` → liste `git remote` e pergunte qual usar. `projeto:"..."` informado explicitamente tem precedência.

Busque o projeto (`GET /projects/<caminho_enc>`) e guarde `PROJECT_ID`, `default_branch` e `web_url`.

### Passo 3 — Branch de destino

`destino:"..."` informado → use-o; senão, `default_branch`. Se `SOURCE_BRANCH` for igual ao destino → informe que não há o que comparar e encerre.

### Passo 4 — O branch está no remoto e atualizado?

Codifique o nome do branch (`/` → `%2F`, `.` → `%2E`):

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/repository/branches/<branch_enc>" \
  -o "<scratchpad>/branch.json" -w "%{http_code}\n"
```

- `404` → o branch não existe no remoto. **Pare** e mostre o comando para o usuário rodar ele mesmo: `git push -u origin <SOURCE_BRANCH>`.
- `commit.id` igual a `LOCAL_SHA` → ok.
- `commit.id` diferente:
  - `git merge-base --is-ancestor <commit.id> HEAD` com sucesso → há commits locais não enviados. **Pare**, mostre `git log --oneline <commit.id>..HEAD` e o comando `git push origin <SOURCE_BRANCH>`.
  - `git merge-base --is-ancestor HEAD <commit.id>` com sucesso → o remoto está à frente do local. Avise que o MR usará o estado do **remoto** e siga.
  - Nenhum dos dois (divergiram, ou o commit remoto não existe localmente) → **pare** e explique; não sugira `--force`.

A skill **nunca** executa `git push`, `git fetch` nem qualquer comando que altere o repositório local.

### Passo 5 — Já existe MR aberto?

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/merge_requests?state=opened&source_branch=<SOURCE_BRANCH>&target_branch=<destino>" \
  -o "<scratchpad>/mr-existente.json"
```

Não vazio → mostre `!iid`, título e `web_url`, informe que já existe e encerre, oferecendo `ler` ou `revisar`.

### Passo 6 — Coletar commits e alterações

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/repository/compare?from=<destino_enc>&to=<source_enc>" \
  -o "<scratchpad>/compare.json"
```

`commits` vazio → não há diferença entre os branches: informe e encerre. Use `commits[].title` e `diffs[]` para entender o que o MR faz.

### Passo 7 — Redigir título e descrição

Siga [mr-models.md](references/mr-models.md). Resumo:

- Título: frase curta (máx. 80 caracteres) + sufixo ` (Criado pelo Claude Code)`. Com `draft:sim`, prefixe `Draft: `.
- Descrição: modelo com Resumo, Alterações, Como testar e Relacionado.
- Número de issue no nome do branch (`123-login`, `feature/123-login`, `issue-123`) → confirme que a issue existe (`GET /projects/:id/issues/123`) e inclua `Relacionado a #123`.

### Passo 8 — Resolver pessoas e labels

- `assignee_id`: `id` de `GET /user` (o dono do token).
- `revisor:"usuario"` → `GET /users?username=<usuario>`; não encontrado → avise e siga sem ele. Guarde em `reviewer_ids`.
- `labels:"..."` → confira em `GET /projects/:id/labels?per_page=100` e envie **só** as que já existem (o GitLab criaria as inexistentes silenciosamente); avise as descartadas.

### Passo 9 — Confirmar

```
Projeto:    <path_with_namespace>
Branches:   <SOURCE_BRANCH> → <destino>
Commits:    <N>
Título:     <título>
Atribuído:  <seu nome>
Revisores:  <nomes ou "nenhum">
Labels:     <labels ou "nenhuma">
Apagar branch de origem no merge: sim

Descrição:
<descrição>
```

`AskUserQuestion`: **Criar MR** ou **Cancelar**.

### Passo 10 — Criar via API

Grave com a tool `Write` em `<scratchpad>/mr-novo.json`:

```json
{
  "source_branch": "<SOURCE_BRANCH>",
  "target_branch": "<destino>",
  "title": "<título>",
  "description": "<descrição em Markdown>",
  "assignee_id": 12,
  "reviewer_ids": [34],
  "labels": "Melhoria",
  "remove_source_branch": true
}
```

Omita `reviewer_ids` e `labels` se vazios.

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/merge_requests" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/mr-novo.json" \
  -o "<scratchpad>/mr-criado.json" -w "%{http_code}\n"
```

### Passo 11 — Reportar

Sucesso (`201`): `!iid`, título, branches e `web_url`. `409` → já existe MR para esses branches (corrida com outra pessoa): mostre-o. Outros erros: exiba `message`/`error` e consulte a tabela em [api-reference.md](references/api-reference.md).
