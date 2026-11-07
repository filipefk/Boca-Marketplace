---
name: azure-card
description: Cria ou lê um work item no Azure DevOps a partir de uma descrição em linguagem natural. Todo card criado contém critérios de aceite, usa as tags já existentes no projeto por afinidade e, quando necessário, é vinculado a outros cards apenas pelo relacionamento Related (não cria cards filhos). A URL do board e a API KEY do usuário são obtidos nas variáveis de ambiente AZURE_URL_BOARD e AZURE_USER_API_KEY
---

## Como usar

```
/azure-card <descrição livre da funcionalidade ou bug>
/azure-card <descrição> relacionado ao card <id>[, <id>...]
/azure-card ler <id>
```

## Referências

- [Regras de criação de cards](references/card-rules.md) — tipo, título, critérios de aceite, tags, relacionamento, confirmação
- [Modelos de cards (HTML)](references/card-models.md) — templates HTML para User Story, Bug e Task

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de um número (ex: `ler 543`) → **fluxo de leitura**.
2. Se a descrição pedir a criação de mais de um card (ex: "uma User Story e duas Tasks", "criar três tasks") → **fluxo de criação de múltiplos cards relacionados**. Se também mencionar IDs de cards existentes, todos os cards criados recebem o relacionamento Related com eles.
3. Caso contrário → **fluxo de criação de card**. Se a descrição mencionar IDs de cards existentes (ex: "relacionado ao 123", "vinculado aos cards 456 e 789", "filho do 321"), o card é criado com relacionamento Related a esses cards.

> **Cards filhos não são suportados.** Pedidos como "filho do card 123" ou "com tasks filhas" são atendidos com o relacionamento **Related** (`System.LinkTypes.Related`), avisando o usuário na confirmação. Nunca usar `System.LinkTypes.Hierarchy-Forward` ou `System.LinkTypes.Hierarchy-Reverse`.

---

## Fluxo de leitura

### Passo 1 — Buscar o card via API

Execute o seguinte script bash:

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

ID="<id extraído dos argumentos>"
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/${ID}?api-version=7.1"

curl -s -H "Authorization: Bearer ${AZURE_USER_API_KEY}" "$URL"
```

### Passo 2 — Exibir os dados do card

Apresente as informações relevantes em formato legível:

```
ID:      <id>
Tipo:    <System.WorkItemType>
Título:  <System.Title>
Estado:  <System.State>
Área:    <System.AreaPath>
Tags:    <System.Tags>
Criado:  <System.CreatedDate>

Descrição:
<System.Description (ou Microsoft.VSTS.TCM.ReproSteps para Bug) sem tags HTML>
```

---

## Buscar tags existentes

Etapa comum aos fluxos de criação. Liste as tags já cadastradas no projeto para escolher as que têm afinidade com o card (regras em [card-rules.md](references/card-rules.md)):

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

URL="${AZURE_URL_BOARD}/_apis/wit/tags?api-version=7.1-preview.1"

curl -s -H "Authorization: Bearer ${AZURE_USER_API_KEY}" "$URL"
```

A resposta traz a lista em `value[].name`. Se a chamada falhar, informe o erro ao usuário e siga sem tags.

---

## Script de criação de card

Etapa comum aos fluxos de criação. Bug usa `Microsoft.VSTS.TCM.ReproSteps`; demais tipos usam `System.Description`.

- Inclua a operação de `System.Tags` somente se houver tags selecionadas.
- Inclua uma operação `/relations/-` com `System.LinkTypes.Related` para **cada** card a relacionar (omita se não houver nenhum).

```bash
AZURE_USER_API_KEY="${AZURE_USER_API_KEY:?Variável AZURE_USER_API_KEY não encontrada}"
AZURE_URL_BOARD="${AZURE_URL_BOARD:?Variável AZURE_URL_BOARD não encontrada}"

WORK_ITEM_TYPE="<tipo do work item, ex: User Story>"
WORK_ITEM_TYPE_ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${WORK_ITEM_TYPE}'))")
URL="${AZURE_URL_BOARD}/_apis/wit/workitems/\$${WORK_ITEM_TYPE_ENCODED}?api-version=7.1"

# Use /fields/Microsoft.VSTS.TCM.ReproSteps para Bug; /fields/System.Description para os demais
DESCRIPTION_PATH="<caminho do campo conforme tipo>"

PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
[
  { "op": "add", "path": "/fields/System.Title", "value": "<título>" },
  { "op": "add", "path": "${DESCRIPTION_PATH}", "value": "<descrição HTML com critérios de aceite>" },
  { "op": "add", "path": "/fields/System.Tags", "value": "<tag1; tag2>" },
  {
    "op": "add",
    "path": "/relations/-",
    "value": {
      "rel": "System.LinkTypes.Related",
      "url": "${AZURE_URL_BOARD}/_apis/wit/workitems/<id do card relacionado>",
      "attributes": { "comment": "" }
    }
  }
]
EOF

RESPONSE=$(curl -s -X POST "$URL" \
  -H "Authorization: Bearer ${AZURE_USER_API_KEY}" \
  -H "Content-Type: application/json-patch+json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}")

rm -f "$PAYLOAD_FILE"

if echo "$RESPONSE" | grep -q '"id"'; then
  echo "Card criado — $RESPONSE"
else
  echo "ERRO ao criar card '<título>': $RESPONSE"
fi
```

> `System.LinkTypes.Related` é um relacionamento simétrico: adicioná-lo no card novo já faz o vínculo aparecer nos dois cards.

---

## Fluxo de criação de card

### Passo 1 — Interpretar a descrição

Leia o texto em `$ARGUMENTS` e infira tipo, título, descrição (sempre com critérios de aceite) conforme as regras em [card-rules.md](references/card-rules.md) e os modelos HTML em [card-models.md](references/card-models.md). Extraia também os IDs de cards existentes a relacionar, se houver.

### Passo 2 — Selecionar tags

Execute [Buscar tags existentes](#buscar-tags-existentes) e escolha as tags com afinidade ao card.

### Passo 3 — Exibir o card para confirmação

Mostre ao usuário o card montado (formato em [card-rules.md](references/card-rules.md)), incluindo tags e cards relacionados. Se o usuário tiver pedido um card "filho", avise que será usado o relacionamento Related.

Use `AskUserQuestion` com as opções **Criar card** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 4 — Criar o card via API

Execute o [Script de criação de card](#script-de-criação-de-card).

### Passo 5 — Reportar o resultado

- **Sucesso:** exibir o ID e a URL do card criado (campo `_links.html.href` da resposta) e os IDs dos cards relacionados, se houver.
- **Erro:** exibir a mensagem de erro retornada pela API.

---

## Fluxo de criação de múltiplos cards relacionados

Use este fluxo quando o usuário pedir mais de um card de uma vez (ex: "User Story com duas Tasks"). Os cards são criados de forma independente — **sem relação pai/filho** — e relacionados entre si via Related.

### Passo 1 — Interpretar os cards

A partir dos `$ARGUMENTS`, infira o **card principal** (normalmente a User Story ou o card mais abrangente) e os **demais cards**, seguindo as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md). Cada card tem seu próprio tipo, título (com o sufixo ` (Criado pela Claudinha)`) e descrição com critérios de aceite. Extraia também IDs de cards existentes a relacionar, se houver.

### Passo 2 — Selecionar tags

Execute [Buscar tags existentes](#buscar-tags-existentes) uma única vez e escolha as tags com afinidade para cada card.

### Passo 3 — Exibir para confirmação

Mostre o plano completo antes de criar qualquer coisa:

```
CARD PRINCIPAL
Tipo:    <tipo>
Título:  <título>
Tags:    <tags ou "(nenhuma)">

Cards relacionados ao principal (Related):
  1. [Task] <título 1> — Tags: <tags>
  2. [Task] <título 2> — Tags: <tags>
  ...

Relacionados também a cards existentes: <IDs ou "(nenhum)">
```

Use `AskUserQuestion` com as opções **Criar todos** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 4 — Criar o card principal

Execute o [Script de criação de card](#script-de-criação-de-card) para o card principal, com relacionamento Related aos cards existentes informados (se houver). Guarde o `$mainId` retornado.

Se a criação do card principal falhar, exiba o erro e encerre sem criar os demais.

### Passo 5 — Criar os demais cards

Para cada card restante, execute o [Script de criação de card](#script-de-criação-de-card) com relacionamento Related ao `$mainId` (e aos cards existentes informados, se houver). Uma falha em um card não interrompe a criação dos demais.

### Passo 6 — Reportar o resultado

Exiba um resumo com os IDs e URLs de todos os cards criados. Se algum falhou, liste-o como erro sem interromper o relatório dos demais:

```
Card principal:
  ID <mainId> — <url>

Cards relacionados criados:
  ID <id1> — <url 1>
  ID <id2> — <url 2>

Cards com erro (se houver):
  <título> — <mensagem de erro>
```
