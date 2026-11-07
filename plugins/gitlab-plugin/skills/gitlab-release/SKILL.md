---
name: gitlab-release
description: Lista, lê, cria, atualiza e apaga releases de um projeto do GitLab (Community Edition, API REST v4), e gerencia os links de assets. Dada uma tag, mostra as mudanças em relação à tag anterior — commits, merge requests (PRs) e cards (issues) envolvidos, e arquivos alterados — e gera as notas da release a partir disso. NÃO cria e NÃO apaga tags (a tag precisa existir; apagar a release mantém a tag). Usa as mesmas credenciais da skill gitlab-card — GITLAB_URL e GITLAB_TOKEN (e opcionalmente GITLAB_PROJECT).
---

## Como usar

```
/gitlab-release listar
/gitlab-release ler <tag | URL da release>
/gitlab-release mudancas <tag> [desde:"v1.2.0"] [arquivos:sim]
/gitlab-release criar <tag> [nome:"..."] [notas:auto | notas:"<texto ou instrução>"] [milestone:"..."] [data:"YYYY-MM-DD"]
/gitlab-release atualizar <tag> [nome:"..."] [notas:auto | notas:"<texto ou instrução>"] [milestone:"..."] [data:"YYYY-MM-DD"]
/gitlab-release apagar <tag>
/gitlab-release links <tag>
/gitlab-release link-adicionar <tag> nome:"Instalador" url:"https://..." [tipo:package|runbook|image|other]
/gitlab-release link-remover <tag> link:<id | nome>
```

Todos aceitam `projeto:"grupo/projeto"`. "Card" é issue e "PR" é merge request (MR), como nas skills `gitlab-card` e `gitlab-pr`.

## ⛔ O que esta skill NÃO faz — nunca execute

| Ação | Por que é proibida | O que **nunca** deve ser chamado |
|---|---|---|
| **Criar tag** | Tag marca um ponto do código — decisão humana, feita pelo git | `POST .../repository/tags`, e o campo **`ref`** no `POST .../releases` (ele cria a tag implicitamente) |
| **Apagar tag** | Irreversível para quem já baixou/usou a versão | `DELETE .../repository/tags/:tag_name` |
| `git tag`, `git push` (qualquer forma) | A skill não altera o repositório local nem o remoto por git | — |

Se o usuário pedir uma dessas ações: **não chame a API**. Diga que está fora do escopo da `gitlab-release` e mostre os comandos para ele rodar:

```bash
git tag -a <tag> <branch-ou-sha> -m "<mensagem>"
git push origin <tag>
```

(para apagar: `git push origin --delete <tag>` e `git tag -d <tag>`).

## Referências

- [Configuração do token](references/token-setup.md) — variáveis de ambiente e permissões (as mesmas da skill `gitlab-card`)
- [Referência da API v4](references/api-reference.md) — endpoints de releases, links, tags, compare, commit→MR, MR→issues; erros e endpoints proibidos
- [Notas da release](references/release-notes.md) — modelo das notas e regras de categorização
- [Lições aprendidas da wiki](../gitlab-wiki/references/licoes-aprendidas.md) — §2 (acento em argumento de linha de comando → `400`) e §6 (dois shells) valem igualmente aqui

## Variáveis de ambiente

| Variável         | Obrigatória | Descrição                                                                    |
|------------------|-------------|------------------------------------------------------------------------------|
| `GITLAB_URL`     | sim         | URL base da instância, sem barra final (ex.: `https://gitlab.olfar.ind.br`) |
| `GITLAB_TOKEN`   | sim         | Personal Access Token com escopo `api` (`read_api` basta para os fluxos de leitura) |
| `GITLAB_PROJECT` | não         | Projeto padrão: caminho `grupo/projeto` ou id numérico                        |

São exatamente as mesmas variáveis usadas pelas skills `gitlab-card`, `gitlab-pr` e `gitlab-wiki`.

Valide no início de cada fluxo:

```bash
GITLAB_URL="${GITLAB_URL:?Variável GITLAB_URL não encontrada — veja references/token-setup.md}"
GITLAB_TOKEN="${GITLAB_TOKEN:?Variável GITLAB_TOKEN não encontrada — veja references/token-setup.md}"
API="${GITLAB_URL%/}/api/v4"
```

Toda chamada é autenticada pelo header `PRIVATE-TOKEN`. **Nunca imprima o valor de `GITLAB_TOKEN`** nem o inclua na URL — deixe sempre o shell expandir a variável dentro do `curl`. Todos os comandos desta skill são **Bash** (Git Bash); não misture sintaxe do PowerShell.

## Conceitos do GitLab

- A release é identificada pela **tag** (`tag_name`), não por um id. Existe no máximo uma release por tag.
- Toda release pertence a uma **tag existente**. Apagar a release **não** apaga a tag nem o código.
- `released_at` no futuro gera uma **upcoming release** (selo "Upcoming" na interface); no passado, uma **historical release**. Sem `released_at`, vale o momento da criação.
- A descrição (`description`) é Markdown. Referências `#12` e `!34` viram links automaticamente.
- `milestones` recebe **títulos** de milestones que precisam já existir no projeto.
- Os **assets** de uma release são os arquivos-fonte gerados automaticamente (`zip`, `tar.gz`...) e **links** cadastrados à mão (pacotes, runbooks, imagens).

## Instruções de execução

Ao ser invocada, verifique `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

0. Pede **criar ou apagar uma tag** → **recusa** (seção ⛔ acima).
1. Começa com `listar` → **fluxo de listagem**.
2. Começa com `ler` (ou `ver`), ou contém uma URL `<GITLAB_URL>/<caminho>/-/releases/<tag>` sem outro verbo → **fluxo de leitura**.
3. Começa com `mudancas`, `mudanças`, `changelog`, `diff` ou `compare` → **fluxo de mudanças**.
4. Começa com `criar` → **fluxo de criação**.
5. Começa com `atualizar` (ou `editar`) → **fluxo de atualização**.
6. Começa com `apagar` (ou `excluir`, `deletar`) → **fluxo de exclusão**.
7. Começa com `link-adicionar` → **fluxo de links** (adicionar).
8. Começa com `link-remover` → **fluxo de links** (remover).
9. Começa com `links` → **fluxo de links** (listar).
10. Nenhum verbo reconhecido → pergunte com `AskUserQuestion` o que fazer, oferecendo as opções mais prováveis para o texto informado (ex.: só uma tag → **Ver mudanças**, **Ler release**, **Criar release**).

---

## Passo comum

### A. Projeto → `PROJECT_ID`

Ordem de precedência:

1. Caminho extraído da URL da release (o trecho entre o host e `/-/releases/`), se houver.
2. Valor do parâmetro `projeto:"..."`.
3. Conteúdo de `$GITLAB_PROJECT`.
4. Nada informado → liste os projetos e pergunte com `AskUserQuestion` (máximo 4 opções; se houver mais, mostre os 4 primeiros e avise que há outros).

Se o valor contiver `/`, é um caminho: use-o direto como `:id`, **URL-encoded** (`grupo/projeto` → `grupo%2Fprojeto`; subgrupos codificam cada barra). Se for numérico, use-o direto. Caso contrário, busque pelo nome:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects?membership=true&simple=true&per_page=100&search=<nome>"
```

Case `name` ou `path_with_namespace` (case-insensitive). Zero resultados → erro e encerre. Mais de um → `AskUserQuestion`. Guarde `id` como `PROJECT_ID`, o `path_with_namespace` e o `web_url`.

### B. Tag → `TAG` e `TAG_ENC`

Aceite:

- **Nome da tag** (`v1.4.0`, `release/2026.09`).
- **URL** `https://gitlab.olfar.ind.br/grupo/projeto/-/releases/v1.4.0` → tudo depois de `/-/releases/` é a tag (sem `?...` e `#...`, e decodificando `%2F` etc.); o trecho antes é o projeto (tem precedência).
- **URL de tag** `.../-/tags/v1.4.0` → mesmo tratamento.

Gere `TAG_ENC` percent-encodando para uso no caminho: `/` → `%2F`, `.` → `%2E`, espaço → `%20`. Valide que a tag existe:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/repository/tags/${TAG_ENC}" \
  -o "<scratchpad>/tag.json" -w "%{http_code}\n"
```

- `200` → guarde `name`, `target` (SHA para onde a tag aponta), `commit.short_id`, `commit.title`, `commit.created_at` e `message` (tag anotada).
- `404` → a tag **não existe**. Busque tags parecidas (`GET .../repository/tags?search=<parte do nome>&per_page=20`), mostre-as e oriente o usuário a criar a tag ele mesmo com os comandos da seção ⛔. **Encerre** — a skill nunca cria tags.

Sem tag informada nos fluxos que exigem uma → liste as 4 tags mais recentes (`order_by=updated&sort=desc&per_page=4`) e pergunte com `AskUserQuestion`.

### C. Tag anterior → `PREV_TAG`

1. `desde:"..."` informado → valide-a como na parte **B** e use-a.
2. Senão, liste as tags em ordem de versão:

   ```bash
   curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     "${API}/projects/${PROJECT_ID}/repository/tags?order_by=version&sort=desc&per_page=100&page=1" \
     -D "<scratchpad>/tags-headers.txt" \
     -o "<scratchpad>/tags-1.json"
   ```

   Localize `TAG` na lista (paginando conforme a parte **E** se ela não estiver na primeira página) e pegue a **próxima** — é a versão imediatamente anterior.
3. Se as tags não seguem versionamento semântico (nomes como `deploy-prod`, datas misturadas com versões) e a ordem por versão parecer sem sentido, use como alternativa a release anterior por data (`GET .../releases?order_by=released_at&sort=desc`) ou `order_by=updated`. Se ainda houver dúvida, pergunte com `AskUserQuestion` mostrando 3–4 candidatas.
4. `TAG` é a primeira tag do projeto → não há anterior. Avise, e pergunte com `AskUserQuestion` se quer comparar desde o **início do repositório** (pode ser enorme) ou informar `desde:`.

**Sempre informe no resultado qual `PREV_TAG` foi usada e como foi escolhida**, para o usuário poder corrigir com `desde:`.

Tags de pré-release (`-rc`, `-beta`, `-alpha`): se `TAG` é uma versão final (`v1.4.0`) e a anterior na lista é um pré-release dela mesma (`v1.4.0-rc2`), prefira a última **versão final** anterior (`v1.3.x`) e avise — as notas de uma versão final costumam cobrir todo o ciclo. Se `TAG` for ela própria um pré-release, use a tag imediatamente anterior.

### D. Enviar texto à API — sempre por arquivo

Todo payload com texto livre (nome, descrição, nome de link) vai num **arquivo JSON gravado com a tool `Write`** (UTF-8) em `<scratchpad>/`, enviado com `--data-binary @arquivo`:

```bash
curl -s -X POST "<endpoint>" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/payload.json" \
  -w "\n%{http_code}\n"
```

Nunca passe texto com acento pela linha de comando (`-d`, `--form "x=..."`): o Git Bash corrompe os bytes e o GitLab devolve `400` — ver [licoes-aprendidas.md §2](../gitlab-wiki/references/licoes-aprendidas.md#2). No JSON, escape `"`, `\` e quebras de linha (`\n`).

### E. Respostas grandes e paginação

Não há `jq` nem `python` no PATH. Salve respostas grandes (compare, listagens) com `-o <scratchpad>/<nome>.json` e leia com a tool `Read`. Envie sempre `per_page=100`; se o header `X-Next-Page` vier preenchido (use `-D <scratchpad>/headers.txt` para inspecionar), repita com `page=N` até esgotar.

### F. A release existe? → `RELEASE`

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}?include_html_description=false" \
  -o "<scratchpad>/release.json" -w "%{http_code}\n"
```

`200` → existe (leia com `Read`). `404` → não existe release para essa tag (a tag pode existir).

---

## Fluxo de listagem

### Passo 1 — Resolver e listar

Parte **A** do Passo comum, depois:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/releases?order_by=released_at&sort=desc&per_page=100" \
  -D "<scratchpad>/releases-headers.txt" \
  -o "<scratchpad>/releases.json"
```

Por padrão mostre as 20 mais recentes; pagine só se o usuário pedir todas.

### Passo 2 — Exibir

```
Releases de <path_with_namespace> — <N>

- <tag_name> — <name>   <[Upcoming] se upcoming_release=true>
  <released_at (data)> · commit <commit.short_id> · <X> link(s) · <_links.self>
```

Lista vazia → informe que o projeto não tem releases e ofereça `mudancas` na tag mais recente.

---

## Fluxo de leitura

### Passo 1 — Resolver e buscar

Partes **A**, **B** e **F** do Passo comum. `404` em **F** com a tag existente → informe que a tag existe mas não tem release, e ofereça `criar` e `mudancas`.

### Passo 2 — Exibir

```
Release:     <name>
Tag:         <tag_name> → commit <commit.short_id> (<commit.title>)
Projeto:     <path_with_namespace>
Publicada:   <released_at><, Upcoming se upcoming_release=true>
Criada em:   <created_at> por <author.name> (@<author.username>)
Milestones:  <milestones[].title ou "nenhum">
Links:       <assets.links[] como "- <name> (<link_type>): <url>", ou "nenhum">
Fontes:      <assets.sources[].format separados por vírgula>
Evidências:  <evidences[].collected_at, ou "nenhuma">
URL:         <_links.self>

Notas:
<description, ou "(sem descrição)">
```

Ao final, ofereça `mudancas` para a mesma tag.

---

## Fluxo de mudanças

Mostra o que entrou numa tag em relação à tag anterior: cards, MRs, commits e arquivos. É também a base das notas geradas por `criar`/`atualizar` com `notas:auto`.

### Passo 1 — Resolver

Partes **A**, **B** e **C** do Passo comum → `TAG`, `PREV_TAG`.

### Passo 2 — Comparar as tags

Codifique as tags em `from`/`to` como na parte **B**:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/repository/compare?from=<PREV_TAG_ENC>&to=<TAG_ENC>" \
  -o "<scratchpad>/compare.json" -w "%{http_code}\n"
```

Sem `straight` (padrão `false`): o GitLab compara a partir do **merge-base**, que é o que interessa ("o que tem em `TAG` que não tinha em `PREV_TAG`").

Do resultado, guarde:

- `commits[]` — `id`, `short_id`, `title`, `message`, `author_name`, `created_at`, `parent_ids` (mais de um pai = merge commit).
- `diffs[]` — `old_path`, `new_path`, `new_file`, `renamed_file`, `deleted_file`, `diff` (conte as linhas que começam com `+`/`-`, ignorando `@@`).
- `compare_timeout: true`, ou `diffs` vazio com commits presentes → o GitLab truncou o diff: avise que a lista de arquivos pode estar incompleta (os commits continuam confiáveis).
- `commits` vazio → as tags apontam para o mesmo código (ou `PREV_TAG` está à frente): informe e encerre.

Se houver mais de ~300 commits, avise do volume antes de seguir e pergunte com `AskUserQuestion` se continua (**Continuar** / **Informar outra tag anterior** / **Cancelar**).

### Passo 3 — MRs envolvidos → `MRS`

Monte o conjunto (sem duplicatas) de `iid`s:

1. **Mensagens de commit.** Nos merge commits padrão do GitLab a mensagem traz `See merge request <grupo/projeto>!<iid>` — extraia o `iid` quando o caminho for o do projeto atual. Também aceite `!<iid>` solto em `title`/`message`. MRs de **outro** projeto (`outro/projeto!12`) → liste à parte, sem buscar.
2. **Commits não cobertos por (1)** — squash merge, fast-forward, ou commits diretos. Para cada commit **que não é merge** e cujo SHA não pertença a um MR já encontrado:

   ```bash
   curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     "${API}/projects/${PROJECT_ID}/repository/commits/<sha>/merge_requests" \
     -o "<scratchpad>/commit-mrs-<short_id>.json"
   ```

   Considere só MRs com `state: merged`. Isso custa uma chamada por commit: acima de ~50 commits restantes, avise e pergunte com `AskUserQuestion` (**Consultar todos** / **Só pelas mensagens de commit**).
3. Para cada `iid` encontrado, busque o MR:

   ```bash
   curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     "${API}/projects/${PROJECT_ID}/merge_requests/<iid>" \
     -o "<scratchpad>/mr-<iid>.json"
   ```

   Guarde `title`, `author.username`, `merged_at`, `labels`, `source_branch`, `target_branch`, `description` e `web_url`. Descarte os que não estão `merged` (avise se algum foi descartado).

Commits que não pertencem a nenhum MR (nem por mensagem nem pela consulta) vão para **Commits sem MR** — descartando merge commits e os que só fazem parte de um MR.

### Passo 4 — Cards envolvidos → `CARDS`

Monte o conjunto de issues, registrando **por onde** cada uma foi encontrada:

1. Issues que o MR fecha:

   ```bash
   curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     "${API}/projects/${PROJECT_ID}/merge_requests/<iid>/closes_issues" \
     -o "<scratchpad>/mr-<iid>-closes.json"
   ```

2. Issues relacionadas ao MR (menções): mesmo formato em `.../merge_requests/<iid>/related_issues`.
3. Referências no texto: `#N`, `Closes #N`, `Fixes #N`, `Resolves #N`, `Relacionado a #N` no título/descrição dos MRs e nas mensagens de commit. Ignore `#N` dentro de blocos de código e referências a outros projetos (`outro/projeto#N`).
4. Número no nome do branch de origem do MR: `123-login`, `feature/123-login`, `issue-123`, `bug/123` → candidato `#123`.

Candidatos vindos de (3) e (4) ainda não confirmados por (1)/(2) → confirme com `GET .../issues/<N>` (`404` = descarte em silêncio). Para cada card guarde `title`, `state`, `labels`, `web_url` e identifique:

- **tipo**: a label de tipo do projeto (ver [release-notes.md](references/release-notes.md) — `Bug`, `Melhoria`, `Segurança`, `Tarefa`...);
- **coluna**: a label de coluna do board (`Backlog`, `To Do`, `Doing`, `Done`), se houver.

### Passo 5 — Exibir

```
Mudanças de <TAG> em relação a <PREV_TAG> — <path_with_namespace>
(tag anterior escolhida por: <desde: | ordem de versão | data da release>)

<N> commits · <M> MRs · <K> cards · <F> arquivos (+<a> −<r>)
Compare: <web_url do projeto>/-/compare/<PREV_TAG>...<TAG>

Cards (<K>)
- #12 [Título](web_url) — Bug · closed · via !34 (fecha)
- #15 [Título](web_url) — Melhoria · opened · Doing · via branch de !36   ⚠ ainda aberto

MRs (<M>)
- !34 [Título](web_url) — @autor · merged em <data> · fecha #12
- !36 [Título](web_url) — @autor · merged em <data> · relacionado a #15

Commits sem MR (<C>)
- abc1234 <title> — <author_name> · <data>

Arquivos alterados (<F>) — por diretório
  src/api/     12 arquivo(s)  (+340 −120)
  docs/         2 arquivo(s)  (+15 −3)
```

- Destaque cards ainda **abertos** (⚠) — entraram na tag mas o card não foi fechado: pode ser trabalho parcial ou card esquecido.
- Lista completa de arquivos (com `A`/`M`/`D`/`R` como no fluxo de alterações da `gitlab-pr`) só com `arquivos:sim` ou se o usuário pedir.
- Seção vazia → omita-a (exceto "Cards", onde vale dizer "nenhum card identificado").

Ao final, com a parte **F** do Passo comum: se a release **não existe**, ofereça `criar <TAG> notas:auto`; se existe, ofereça `atualizar <TAG> notas:auto`.

---

## Fluxo de criação

### Passo 1 — Extrair parâmetros e resolver

De `$ARGUMENTS` (após `criar`): a tag, `nome`, `notas`, `milestone` (aceita vários, separados por vírgula), `data`, `projeto`. Partes **A** e **B** do Passo comum — **tag inexistente encerra o fluxo** (a skill nunca cria tags).

### Passo 2 — Já existe release?

Parte **F**. `200` → a release já existe: mostre nome e `_links.self` e pergunte com `AskUserQuestion`: **Atualizar a existente** (segue para o fluxo de atualização) ou **Cancelar**.

### Passo 3 — Montar os campos

- **Nome** (`name`): `nome:` informado, senão o próprio `TAG`.
- **Notas** (`description`):
  - `notas:auto` **ou nada informado** → execute o fluxo de mudanças (Passos 1 a 4) e redija conforme [release-notes.md](references/release-notes.md).
  - `notas:"..."` com texto pronto → use como está.
  - `notas:"..."` com uma instrução ("resuma só as correções", "em inglês") → execute o fluxo de mudanças e redija seguindo a instrução.
- **Milestones**: para cada título informado,

  ```bash
  curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
    "${API}/projects/${PROJECT_ID}/milestones?title=<titulo_enc>" \
    -o "<scratchpad>/milestone.json"
  ```

  Envie **só** os que existem; avise os descartados. Sem `milestone:` informado, verifique se existe um milestone com o mesmo título da tag (com e sem o `v` inicial) e, se existir, **sugira** incluí-lo na confirmação — não inclua sem perguntar.
- **Data** (`released_at`): `data:"YYYY-MM-DD"` → `YYYY-MM-DDT00:00:00Z`; aceite também data e hora ISO 8601 completas. Data futura → avise que a release aparecerá como **Upcoming** até lá. Sem `data:`, omita o campo (vale o momento da criação).

### Passo 4 — Confirmar

```
Projeto:     <path_with_namespace>
Tag:         <TAG> → commit <short_id> (<commit.title>)
Nome:        <name>
Milestones:  <títulos ou "nenhum">
Data:        <released_at ou "agora"><, Upcoming se futura>
Base:        mudanças desde <PREV_TAG> (<N> commits, <M> MRs, <K> cards)   ← só se notas:auto

Notas:
<description>
```

`AskUserQuestion`: **Criar release**, **Ajustar notas** (peça o ajuste, reescreva e confirme de novo) ou **Cancelar**. Deixe claro que a release fica visível para todos com acesso ao projeto e pode notificar quem acompanha releases.

### Passo 5 — Criar via API

Grave com a tool `Write` em `<scratchpad>/release-nova.json`:

```json
{
  "tag_name": "<TAG>",
  "name": "<nome>",
  "description": "<notas em Markdown>",
  "milestones": ["<título>"],
  "released_at": "2026-09-25T00:00:00Z"
}
```

Omita `milestones` e `released_at` se não houver. **Nunca inclua `ref`** — com `ref`, o GitLab criaria a tag se ela não existisse.

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/releases" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/release-nova.json" \
  -o "<scratchpad>/release-criada.json" -w "%{http_code}\n"
```

### Passo 6 — Reportar

Sucesso (`201`): tag, nome, milestones e `_links.self`. `409` → já existe release para a tag (corrida com outra pessoa): mostre-a. Outros erros: exiba `message`/`error` e consulte a tabela em [api-reference.md](references/api-reference.md). Ofereça `link-adicionar` se houver pacotes/artefatos a vincular.

---

## Fluxo de atualização

Altera nome, notas, milestones ou data de uma release existente.

### Passo 1 — Resolver e ler a release atual

Partes **A**, **B** e **F** do Passo comum. `404` em **F** → a release não existe: ofereça `criar`.

Sem nenhum campo informado → pergunte com `AskUserQuestion` (multiSelect) o que alterar: **Nome**, **Notas**, **Milestones**, **Data**.

### Passo 2 — Montar os novos valores

- **Notas**:
  - `notas:auto` → regenere pelo fluxo de mudanças, como na criação. Se a descrição atual tem trechos escritos à mão que não vêm das mudanças (avisos, instruções de upgrade), **preserve-os** e avise.
  - Alteração pontual ("acrescente um aviso de breaking change", "remova o item !36") → parta da descrição **atual** e aplique a mudança. O `PUT` substitui a descrição inteira — enviar só o trecho novo apagaria o resto.
  - Texto pronto → substitui tudo (avise).
- **Milestones**: valide como na criação. O campo **substitui** a lista inteira; `milestones: []` remove todos — confirme explicitamente se for o caso.
- **Data**: como na criação.

### Passo 3 — Confirmar

Exiba o antes e o depois de cada campo alterado (para as notas, um resumo das seções adicionadas/removidas/alteradas). `AskUserQuestion`: **Atualizar release** ou **Cancelar**.

### Passo 4 — Atualizar via API

JSON com **só** os campos alterados (`name`, `description`, `milestones`, `released_at`), gravado com `Write`:

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@<scratchpad>/release-update.json" \
  -o "<scratchpad>/release-atualizada.json" -w "%{http_code}\n"
```

A tag de uma release **não pode** ser trocada pelo `PUT` — para "mudar a tag", é preciso apagar a release e criá-la na outra tag (avise o usuário e siga os dois fluxos, com confirmação de cada um).

### Passo 5 — Reportar

Sucesso (`200`): o que foi alterado e `_links.self`.

---

## Fluxo de exclusão

### Passo 1 — Resolver e mostrar

Partes **A**, **B** e **F** do Passo comum. **Leia a release antes** e exiba nome, tag, data, milestones, links e as primeiras linhas das notas — nunca apague às cegas. `404` → informe que não há release para essa tag e encerre.

### Passo 2 — Oferecer backup e confirmar

Ofereça exportar as notas para um arquivo local (`release-<tag com / virando _>.md` no diretório de trabalho, gravado com `Write`), já que elas se perdem com a release.

`AskUserQuestion`: **Apagar release** ou **Cancelar**. Deixe explícito que:

- **a tag `<TAG>` continua existindo** (a skill não apaga tags);
- os links de assets cadastrados são apagados junto com a release;
- a ação não pode ser desfeita pela API.

### Passo 3 — Apagar via API

```bash
curl -s -X DELETE "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -o "<scratchpad>/release-apagada.json" -w "%{http_code}\n"
```

Sucesso é `200` (o corpo traz a release apagada). Reporte o que foi apagado e reforce que a tag foi mantida. Se o usuário pedir para apagar a tag também → seção ⛔.

---

## Fluxo de links

Links de assets da release (pacotes, instaladores, runbooks, imagens).

### Listar

Partes **A**, **B** e **F**; depois:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}/assets/links" \
  -o "<scratchpad>/links.json"
```

```
Links de <TAG> — <N>
- [<id>] <name> (<link_type>) — <url>   <direct_asset_url, se houver>
```

### Adicionar

1. De `$ARGUMENTS`: `nome`, `url`, `tipo` (`other` por padrão; válidos: `other`, `runbook`, `image`, `package`) e, opcionalmente, `caminho:"/binaries/app.zip"` (`direct_asset_path`, gera uma URL permanente `<_links.self>/downloads/binaries/app.zip`). Sem `nome` ou `url` → peça com `AskUserQuestion`.
2. Valide que `url` começa com `http://`, `https://` ou `ftp://`. Liste os links atuais: já existe um com a mesma `url` ou o mesmo `nome` → avise (o GitLab rejeita URL duplicada na mesma release).
3. Confirme (`AskUserQuestion`: **Adicionar link** ou **Cancelar**).
4. JSON `{"name": "...", "url": "...", "link_type": "package", "direct_asset_path": "/..."}` gravado com `Write` (omita `direct_asset_path` se não houver):

   ```bash
   curl -s -X POST "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}/assets/links" \
     -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     -H "Content-Type: application/json; charset=utf-8" \
     --data-binary "@<scratchpad>/link-novo.json" \
     -w "\n%{http_code}\n"
   ```

   Sucesso é `201`.

### Remover

1. Liste os links e resolva `link:` pelo `id` ou pelo `name` (case-insensitive). Mais de um candidato ou nenhum informado → `AskUserQuestion`.
2. Exiba o link e confirme (**Remover link** ou **Cancelar**).
3. Apague:

   ```bash
   curl -s -X DELETE "${API}/projects/${PROJECT_ID}/releases/${TAG_ENC}/assets/links/<link_id>" \
     -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
     -w "\n%{http_code}\n"
   ```

   Sucesso é `200`. O arquivo apontado pela URL não é afetado — só o link na release.
