---
name: gitlab-card
description: Cria, lê, atualiza, move, fecha, reabre, comenta e lista issues no board de um projeto do GitLab (Community Edition, API REST v4) a partir de linguagem natural. Projeto, board e coluna são resolvidos por nome; colunas do board são labels. Também gerencia checklists via task list Markdown. Requer GITLAB_URL e GITLAB_TOKEN (e opcionalmente GITLAB_PROJECT).
---

## Como usar

```
/gitlab-card <descrição livre da funcionalidade ou bug>
/gitlab-card projeto:"grupo/projeto" <descrição>
/gitlab-card projeto:"grupo/projeto" board:"Development" coluna:"Doing" <descrição>
/gitlab-card coluna:"Doing" due:"2026-09-01" <descrição>
/gitlab-card checklist:"Tarefas" <descrição com itens>
/gitlab-card ler <iid | #iid | URL da issue>
/gitlab-card listar coluna:"Doing"
/gitlab-card colunas
/gitlab-card atualizar <ref> <novo título ou descrição>
/gitlab-card mover <ref> coluna:"Doing"
/gitlab-card fechar <ref>
/gitlab-card reabrir <ref>
/gitlab-card comentarios <ref>
/gitlab-card comentar <ref> <texto do comentário>
```

## Referências

- [Configuração do token](references/token-setup.md) — como gerar o Personal Access Token e definir as variáveis
- [Regras de criação de cards](references/card-rules.md) — tipo, título, labels, boas práticas de conteúdo, confirmação
- [Modelos de cards (Markdown)](references/card-models.md) — templates para Story, Bug, Task e checklist
- [Referência da API v4](references/api-reference.md) — endpoints, campos, paginação e códigos de erro

## Variáveis de ambiente

| Variável         | Obrigatória | Descrição                                                                     |
|------------------|-------------|-------------------------------------------------------------------------------|
| `GITLAB_URL`     | sim         | URL base da instância, sem barra final (ex.: `https://gitlab.olfar.ind.br`)  |
| `GITLAB_TOKEN`   | sim         | Personal Access Token com escopo `api`                                        |
| `GITLAB_PROJECT` | não         | Projeto padrão: caminho `grupo/projeto` ou id numérico                         |

Valide as variáveis no início de cada fluxo:

```bash
GITLAB_URL="${GITLAB_URL:?Variável GITLAB_URL não encontrada — veja references/token-setup.md}"
GITLAB_TOKEN="${GITLAB_TOKEN:?Variável GITLAB_TOKEN não encontrada — veja references/token-setup.md}"
API="${GITLAB_URL%/}/api/v4"
```

Toda chamada é autenticada pelo header `PRIVATE-TOKEN`. **Nunca imprima o valor de `GITLAB_TOKEN`** nem o inclua na URL — deixe sempre o shell expandir a variável dentro do `curl`.

## Conceitos do GitLab que mudam em relação ao Trello

- A issue é identificada pelo **`iid`** (número visível, ex.: `#42`), não pelo `id` global. Os endpoints de projeto usam `iid`.
- Uma **coluna do board é uma label**. "Estar na coluna Doing" significa "ter a label Doing". Mover entre colunas = trocar labels.
- As colunas **Open** e **Closed** são implícitas e **não** são retornadas por `GET .../lists`.
- **Não existe arquivar issue.** O equivalente ao arquivar do Trello é **fechar** (`state_event: close`).
- **Não existe objeto de checklist.** Checklist é uma task list Markdown (`- [ ] item`) dentro da descrição.

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de uma referência → **fluxo de leitura**.
2. Se contiver uma URL de issue no formato `<GITLAB_URL>/<caminho>/-/issues/<iid>` (sem prefixo) → **fluxo de leitura**.
3. Se começar com `listar` → **fluxo de listagem de issues**.
4. Se começar com `colunas` → **fluxo de listagem de colunas**.
5. Se começar com `atualizar` → **fluxo de atualização**.
6. Se começar com `mover` → **fluxo de movimentação**.
7. Se começar com `fechar` → **fluxo de fechamento**.
8. Se começar com `reabrir` → **fluxo de reabertura**.
9. Se começar com `comentarios` → **fluxo de listagem de comentários**.
10. Se começar com `comentar` → **fluxo de novo comentário**.
11. Caso contrário → **fluxo de criação**.

---

## Passo comum — Resolver referência, projeto, board e coluna

Todos os fluxos usam esta seção. Execute apenas as partes necessárias ao fluxo em questão.

### A. Referência da issue (`ref`)

Aceite três formas:

- `123` ou `#123` → `ISSUE_IID=123`
- URL `https://gitlab.olfar.ind.br/grupo/projeto/-/issues/123` → extraia `ISSUE_IID` do último segmento **e** o caminho do projeto do trecho entre o host e `/-/issues/`. Esse caminho **tem precedência** sobre `projeto:` e sobre `GITLAB_PROJECT`.

### B. Projeto → `PROJECT_ID`

Determine o projeto nesta ordem de precedência:

1. Caminho extraído da URL da issue, se houver.
2. Valor do parâmetro `projeto:"..."`.
3. Conteúdo de `$GITLAB_PROJECT`.
4. Nada informado → liste os projetos e pergunte com `AskUserQuestion` (máximo 4 opções; se houver mais, mostre os 4 primeiros e avise que há outros).

Se o valor contiver `/`, é um caminho: use-o direto como `:id`, **URL-encoded** (`grupo/projeto` → `grupo%2Fprojeto`). Se for um id numérico, use-o direto. Caso contrário, busque pelo nome:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects?membership=true&simple=true&per_page=100&search=<nome>"
```

Case `name` ou `path_with_namespace` (comparação case-insensitive). Zero resultados → informe o erro e encerre. Mais de um resultado → use `AskUserQuestion` para o usuário escolher. Guarde `id` como `PROJECT_ID` e o `web_url` do projeto.

### C. Board → `BOARD_ID`

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" "${API}/projects/${PROJECT_ID}/boards"
```

- Com `board:` informado: case por `name` (case-insensitive). Não encontrou → erro listando os boards disponíveis.
- Sem `board:`: use o **primeiro** board retornado.
- Projeto sem nenhum board: avise o usuário e siga sem coluna (a issue nasce em *Open*).

### D. Coluna → `COLUNA_LABEL` e `LABELS_DO_BOARD`

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/boards/${BOARD_ID}/lists"
```

Cada item traz `id`, `position` e `label.name`. Guarde **todos** os `label.name` como `LABELS_DO_BOARD` — essa lista é necessária no fluxo de movimentação.

- Com `coluna:` informado: case `label.name` (case-insensitive). Não encontrou → erro listando as colunas válidas.
- Sem `coluna:` no fluxo de criação: procure em `LABELS_DO_BOARD` uma coluna chamada `Backlog` (case-insensitive) e use-a.
  - Se **não existir** coluna `Backlog`, **não** assuma a coluna de menor `position` nem crie a issue sem confirmar: liste as colunas existentes do board e use `AskUserQuestion` para o usuário escolher em qual colocar o(s) card(s). Aplique a escolha a todos os cards da solicitação atual.

Lembre-se de que *Open* e *Closed* não aparecem nesta resposta.

---

## Fluxo de criação

### Passo 1 — Extrair parâmetros dos argumentos

Analise `$ARGUMENTS` e extraia, se presentes:

- `projeto`: valor de `projeto:"..."` ou `projeto:...`
- `board`: valor de `board:"..."` ou `board:...`
- `coluna`: valor de `coluna:"..."` ou `coluna:...`
- `checklist`: valor de `checklist:"..."` ou `checklist:...` — nome da seção de checklist na descrição
- `due`: valor de `due:"..."` ou `due:...` — data no formato `YYYY-MM-DD` (formato nativo do campo `due_date` do GitLab; **não** converta para ISO 8601 completo)
- O restante do texto (sem os parâmetros acima) é a **descrição do card**

Exemplos:
- `projeto:"infra/api" bug no login` → projeto=`infra/api`, descrição=`bug no login`
- `coluna:"Doing" nova feature` → coluna=`Doing`, descrição=`nova feature`
- `checklist:"Tarefas" nova feature com subtarefas` → checklist=`Tarefas`, descrição=`nova feature com subtarefas`

**Inferência de itens de checklist:** Se `checklist` foi informado, analise a descrição e infira a lista de itens. Os itens podem estar explícitos (ex.: "- item 1", "1. item 1") ou ser inferidos do contexto (critérios de aceite de uma Story, passos de uma Task). Se nada puder ser inferido com segurança, pergunte com `AskUserQuestion`.

### Passo 2 — Resolver projeto, board e coluna

Execute as partes **B**, **C** e **D** do Passo comum.

### Passo 3 — Interpretar a descrição

Infira tipo, título e descrição conforme as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

Se `checklist` foi informado, acrescente ao final da descrição uma seção com o nome da checklist e os itens como task list:

```markdown
## <nome da checklist>
- [ ] <item 1>
- [ ] <item 2>
```

### Passo 4 — Resolver a label de tipo

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/labels?per_page=100"
```

Procure a label cujo `name` corresponda ao tipo inferido (Bug, Story ou Task), comparação case-insensitive. Se **não existir**, crie a issue **sem** a label de tipo e avise no relatório final — não crie labels novas no projeto automaticamente (o GitLab cria labels inexistentes silenciosamente ao receber `labels`, o que poluiria o projeto).

### Passo 5 — Exibir para confirmação

Mostre o card montado (formato em [card-rules.md](references/card-rules.md)), incluindo projeto, board e coluna de destino. Se `due` foi informado, exiba o prazo. Se `checklist` foi informado, exiba o nome e os itens:

```
Checklist: <nome da checklist>
  - <item 1>
  - <item 2>
```

Use `AskUserQuestion` com as opções **Criar issue** e **Cancelar**. Se o usuário cancelar, encerre sem fazer nada.

### Passo 6 — Criar a issue via API

```bash
PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
{
  "title": "<título do card>",
  "description": "<descrição em Markdown>",
  "labels": "<label da coluna>,<label de tipo>"
}
EOF

# "labels" é uma string separada por vírgula.
# Omita a label de coluna se o board não tiver colunas.
# Omita a label de tipo se ela não existir no projeto (Passo 4).
# Acrescente "due_date": "YYYY-MM-DD" apenas se due: foi informado.

curl -s -X POST "${API}/projects/${PROJECT_ID}/issues" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 7 — Reportar o resultado

- **Sucesso:** exiba projeto, board, coluna, `iid`, título e `web_url` da issue criada. Se houve checklist, informe a quantidade de itens. Se a label de tipo não existia, avise.
- **Erro:** exiba a mensagem retornada pela API (campo `message` ou `error`) — consulte a tabela de erros em [api-reference.md](references/api-reference.md).

---

## Fluxo de leitura

### Passo 1 — Resolver referência e projeto

Execute as partes **A** e **B** do Passo comum.

### Passo 2 — Buscar a issue

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}"
```

### Passo 3 — Identificar a coluna

Execute as partes **C** e **D** do Passo comum para obter `LABELS_DO_BOARD` e descubra qual delas está presente em `labels` da issue — essa é a coluna atual. Se nenhuma estiver, a issue está em *Open* (ou *Closed*, se `state` for `closed`).

### Passo 4 — Exibir os dados

```
Issue:     #<iid> — <title>
Projeto:   <path_with_namespace>
Estado:    <state>
Coluna:    <coluna identificada ou "Open">
URL:       <web_url>
Autor:     <author.name>
Atribuída: <assignees[].name ou "sem responsável">
Labels:    <labels separadas por vírgula>
Milestone: <milestone.title ou "sem milestone">
Vence em:  <due_date ou "sem prazo">

Descrição:
<description>
```

---

## Fluxo de listagem de issues

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `listar`) e extraia `projeto`, `board` e `coluna`.

### Passo 2 — Resolver projeto, board e coluna

Execute as partes **B**, **C** e **D** do Passo comum. Se `coluna` não for informada, liste todas as issues abertas do projeto.

### Passo 3 — Listar as issues

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/issues?state=opened&per_page=100&labels=<COLUNA_LABEL>"
```

Omita o parâmetro `labels` se nenhuma coluna foi informada.

### Passo 4 — Exibir as issues

Para cada issue retornada:

```
- #<iid> [<title>](<web_url>) — Labels: <labels> | Vence: <due_date ou "sem prazo">
```

Se não houver issues, informe que a coluna está vazia.

---

## Fluxo de listagem de colunas

Mostra as colunas configuradas no board.

### Passo 1 — Resolver projeto e board

Execute as partes **B**, **C** e **D** do Passo comum.

### Passo 2 — Exibir as colunas

Liste as colunas na ordem de `position`:

```
Board: <nome do board> (projeto <path_with_namespace>)
  1. <label.name>
  2. <label.name>
```

Acrescente a nota de que as colunas **Open** e **Closed** também existem no board, mas são implícitas e não aparecem na API.

---

## Fluxo de atualização

Atualiza o título e/ou a descrição de uma issue existente.

### Passo 1 — Extrair a referência e o novo conteúdo

Analise `$ARGUMENTS` (após remover o prefixo `atualizar`). Execute as partes **A** e **B** do Passo comum. O restante do texto é o novo conteúdo.

Use `AskUserQuestion` para perguntar o que alterar: **Título**, **Descrição** ou **Ambos**.

### Passo 2 — Interpretar o novo conteúdo

- Título: siga as mesmas regras do fluxo de criação (máximo 80 caracteres, sufixo ` (Criado pelo Claude Code)`).
- Descrição: use os modelos em [card-models.md](references/card-models.md). Se a issue já tiver uma seção de checklist e ela não for o alvo da alteração, preserve-a (leia a descrição atual antes, com o Passo 2 do fluxo de leitura).

### Passo 3 — Confirmar com o usuário

Exiba os novos valores e use `AskUserQuestion` com as opções **Atualizar** e **Cancelar**.

### Passo 4 — Atualizar via API

```bash
PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
{
  "title": "<novo título, se alterado>",
  "description": "<nova descrição, se alterada>"
}
EOF

curl -s -X PUT "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

Inclua no JSON apenas os campos alterados.

### Passo 5 — Reportar o resultado

Exiba o `iid` e a `web_url` da issue, informando o que foi alterado.

---

## Fluxo de movimentação

Move a issue para outra coluna do board trocando as labels de coluna.

### Passo 1 — Extrair parâmetros e resolver

Analise `$ARGUMENTS` (após remover o prefixo `mover`). Execute as partes **A**, **B**, **C** e **D** do Passo comum para obter `ISSUE_IID`, `PROJECT_ID`, `COLUNA_LABEL` (destino) e `LABELS_DO_BOARD`.

Se `coluna` não for informada, use `AskUserQuestion` listando as colunas do board.

### Passo 2 — Descobrir as labels atuais da issue

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}"
```

Calcule a **interseção** entre `labels` da issue e `LABELS_DO_BOARD`, **excluindo** a coluna de destino. Esse é o conjunto de labels a remover. Labels que não pertencem ao board (tipo, prioridade, etc.) **nunca** devem ser removidas.

### Passo 3 — Mover via API

```bash
MOVE_PAYLOAD=$(mktemp --suffix=.json)

cat > "$MOVE_PAYLOAD" <<EOF
{
  "add_labels": "${COLUNA_LABEL}",
  "remove_labels": "<labels da interseção, separadas por vírgula>"
}
EOF

curl -s -X PUT "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${MOVE_PAYLOAD}"

rm -f "$MOVE_PAYLOAD"
```

Se a interseção for vazia, omita `remove_labels` do JSON.

### Passo 4 — Reportar o resultado

Informe a coluna de origem, a coluna de destino, o `iid` e a `web_url` da issue.

---

## Fluxo de fechamento

Fecha uma issue. É o equivalente ao arquivar do Trello — o GitLab não arquiva issues.

### Passo 1 — Resolver referência e projeto

Execute as partes **A** e **B** do Passo comum.

### Passo 2 — Confirmar com o usuário

Busque o título da issue (Passo 2 do fluxo de leitura) e exiba-o. Use `AskUserQuestion` com as opções **Fechar issue** e **Cancelar**.

### Passo 3 — Fechar via API

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"state_event": "close"}'
```

### Passo 4 — Reportar o resultado

Informe que a issue foi fechada, exibindo `iid`, título e `web_url`.

---

## Fluxo de reabertura

Idêntico ao fluxo de fechamento, com duas diferenças:

- A confirmação usa as opções **Reabrir issue** e **Cancelar**.
- O payload é `{"state_event": "reopen"}`.

---

## Fluxo de listagem de comentários

### Passo 1 — Resolver referência e projeto

Execute as partes **A** e **B** do Passo comum.

### Passo 2 — Buscar as notes

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}/notes?sort=asc&per_page=100"
```

### Passo 3 — Exibir os comentários

**Descarte todos os itens com `"system": true`** — são eventos automáticos do GitLab (mudou label, fechou, atribuiu) e não comentários de pessoas.

Para cada comentário restante:

```
[<created_at>] <author.name>:
<body>
```

Se não restar nenhum, informe que a issue não tem comentários.

---

## Fluxo de novo comentário

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `comentar`). Execute as partes **A** e **B** do Passo comum. O restante do texto é o corpo do comentário.

Se o texto não for informado, solicite-o com `AskUserQuestion`.

### Passo 2 — Confirmar com o usuário

Exiba o `iid`, o título da issue e o texto do comentário. Use `AskUserQuestion` com as opções **Comentar** e **Cancelar**.

### Passo 3 — Publicar via API

```bash
PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
{"body": "<texto do comentário>"}
EOF

curl -s -X POST "${API}/projects/${PROJECT_ID}/issues/${ISSUE_IID}/notes" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 4 — Reportar o resultado

Informe que o comentário foi publicado, exibindo o `iid` da issue e a `web_url`.
