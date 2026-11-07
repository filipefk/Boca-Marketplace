---
name: trello-card
description: Cria, lê, atualiza, move, arquiva e lista cards no Trello a partir de linguagem natural. Também gerencia checklists. Requer apenas TRELLO_API_KEY e TRELLO_TOKEN. Board e coluna são resolvidos por nome.
---

## Como usar

```
/trello-card <descrição livre da funcionalidade ou bug>
/trello-card board:"Nome do Board" <descrição>
/trello-card board:"Nome do Board" coluna:"Nome da Coluna" <descrição>
/trello-card board:"Nome do Board" coluna:"Nome da Coluna" due:"2026-07-01" <descrição>
/trello-card board:"Nome do Board" checklist:"Nome da Checklist" <descrição com itens>
/trello-card ler <id ou URL>
/trello-card listar board:"Nome do Board" coluna:"Nome da Coluna"
/trello-card atualizar <id ou URL> <novo título ou descrição>
/trello-card mover <id ou URL> board:"Nome do Board" coluna:"Nome da Coluna"
/trello-card arquivar <id ou URL>
/trello-card checklists <id ou URL>
/trello-card checklist-item <id-checklist> <texto do item>
```

## Referências

- [Regras de criação de cards](references/card-rules.md) — tipo, título, labels, confirmação
- [Modelos de cards (Markdown)](references/card-models.md) — templates para Story, Bug e Task

## Variáveis de ambiente

| Variável         | Descrição                        |
|------------------|----------------------------------|
| `TRELLO_API_KEY` | Chave de API do Trello           |
| `TRELLO_TOKEN`   | Token de autenticação do Trello  |

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Se começar com `ler` seguido de um ID ou URL → **fluxo de leitura**.
2. Se contiver uma URL do Trello no formato `https://trello.com/c/<id>/...` (sem prefixo) → **fluxo de leitura**, extraindo o `<id>` da URL.
3. Se começar com `listar` → **fluxo de listagem de cards**.
4. Se começar com `atualizar` → **fluxo de atualização**.
5. Se começar com `mover` → **fluxo de movimentação**.
6. Se começar com `arquivar` → **fluxo de arquivamento**.
7. Se começar com `checklists` → **fluxo de listagem de checklists**.
8. Se começar com `checklist-item` → **fluxo de adição de item a checklist**.
9. Caso contrário → **fluxo de criação**.

---

## Fluxo de leitura

### Passo 1 — Extrair o ID do card e buscar via API

Se o argumento for uma URL do Trello (ex: `https://trello.com/c/IrWu6L3U/4-nome-do-card`), extraia o segmento após `/c/` e antes da próxima `/` como `CARD_ID`. Se for um ID simples (ex: após `ler abc123`), use-o diretamente.

```bash
TRELLO_API_KEY="${TRELLO_API_KEY:?Variável TRELLO_API_KEY não encontrada}"
TRELLO_TOKEN="${TRELLO_TOKEN:?Variável TRELLO_TOKEN não encontrada}"

# Exemplo de extração de ID a partir de URL:
# URL="https://trello.com/c/IrWu6L3U/4-nome-do-card"
# CARD_ID=$(echo "$URL" | sed 's|.*/c/\([^/]*\)/.*|\1|')

CARD_ID="<id extraído dos argumentos ou da URL>"
curl -s "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&fields=name,desc,url,idList,labels,due,dateLastActivity&members=true"
```

### Passo 2 — Exibir os dados do card

```
ID:       <id>
Título:   <name>
URL:      <url>
Labels:   <labels[].name separados por vírgula>
Vence em: <due ou "sem prazo">

Descrição:
<desc>
```

---

## Fluxo de criação

### Passo 1 — Extrair parâmetros dos argumentos

Analise `$ARGUMENTS` e extraia, se presentes:

- `board`: valor do parâmetro `board:"..."` ou `board:...` (sem aspas se for uma só palavra)
- `coluna`: valor do parâmetro `coluna:"..."` ou `coluna:...`
- `checklist`: valor do parâmetro `checklist:"..."` ou `checklist:...` — nome da checklist a ser criada no card
- `due`: valor do parâmetro `due:"..."` ou `due:...` — data de vencimento no formato `YYYY-MM-DD` ou ISO 8601 completo (ex: `2026-07-01` ou `2026-07-01T12:00:00Z`). Se apenas a data for informada, adicione `T00:00:00Z` ao final.
- O restante do texto (sem os parâmetros acima) é a **descrição do card**

Exemplos:
- `board:"Meu Projeto" bug no login` → board=`Meu Projeto`, descrição=`bug no login`
- `bug no login` → board não informado, descrição=`bug no login`
- `board:"Meu Projeto" coluna:"Em andamento" nova feature` → board=`Meu Projeto`, coluna=`Em andamento`, descrição=`nova feature`
- `board:"Meu Projeto" checklist:"Tarefas" nova feature com subtarefas` → board=`Meu Projeto`, checklist=`Tarefas`, descrição=`nova feature com subtarefas`

**Inferência de itens de checklist:** Se `checklist` foi informado, analise a descrição e infira uma lista de itens para a checklist. Itens podem estar explícitos na descrição (ex: "- item 1", "1. item 1") ou podem ser inferidos a partir do contexto (ex: critérios de aceite de uma Story, passos de uma Task). Se nenhum item puder ser inferido com segurança, pergunte ao usuário com `AskUserQuestion`.

### Passo 2 — Resolver o board

**Se o board foi informado:**

```bash
TRELLO_API_KEY="${TRELLO_API_KEY:?Variável TRELLO_API_KEY não encontrada}"
TRELLO_TOKEN="${TRELLO_TOKEN:?Variável TRELLO_TOKEN não encontrada}"

curl -s "https://api.trello.com/1/members/me/boards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&filter=open&fields=id,name"
```

Busque na resposta o board cujo `name` corresponda ao valor informado (comparação case-insensitive). Se não encontrar, informe o erro e encerre.

**Se o board NÃO foi informado:**

Execute o mesmo curl acima e liste os boards encontrados para o usuário escolher usando `AskUserQuestion`. Monte as opções com os nomes dos boards retornados (máximo 4; se houver mais, liste os 4 primeiros e informe que há outros). Após a escolha, use o `id` do board selecionado.

### Passo 3 — Resolver a coluna

Com o `BOARD_ID` obtido no passo anterior:

```bash
curl -s "https://api.trello.com/1/boards/${BOARD_ID}/lists?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&filter=open&fields=id,name,pos"
```

- **Se a coluna foi informada:** encontre na resposta a lista cujo `name` corresponda (case-insensitive). Se não encontrar, informe o erro e encerre.
- **Se a coluna NÃO foi informada:** use a lista com o menor valor de `pos` (a primeira coluna).

Guarde o `id` da lista encontrada como `LIST_ID`.

### Passo 4 — Interpretar a descrição

Leia o texto da descrição extraída no Passo 1 e infira tipo, título e descrição conforme as regras em [card-rules.md](references/card-rules.md) e os modelos em [card-models.md](references/card-models.md).

### Passo 5 — Exibir o card para confirmação

Mostre ao usuário o card montado (formato em [card-rules.md](references/card-rules.md)), incluindo o nome do board e da coluna onde será criado. Se `due` foi informado, exiba também a data de vencimento. Se `checklist` foi informado, exiba também o nome da checklist e seus itens inferidos no formato:

```
Checklist: <nome da checklist>
  - <item 1>
  - <item 2>
  - ...
```

Use `AskUserQuestion` com as opções **Criar card** e **Cancelar**.

Se o usuário cancelar, encerre sem fazer nada.

### Passo 6 — Buscar o ID do label via API

```bash
curl -s "https://api.trello.com/1/boards/${BOARD_ID}/labels?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}"
```

Identifique o label cujo `name` corresponda ao tipo inferido (Bug, Story ou Task) ou cuja `color` corresponda à cor definida nas regras. Se não encontrar label correspondente, crie o card **sem label**.

### Passo 7 — Criar o card via API

```bash
PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
{
  "name": "<título do card>",
  "desc": "<descrição em Markdown>",
  "idList": "${LIST_ID}"
}
EOF

# Adicionar "idLabels": ["<idLabel>"] ao JSON acima somente se encontrou o label no Passo 6
# Adicionar "due": "<valor ISO 8601>" ao JSON acima somente se o parâmetro due foi informado

curl -s -X POST "https://api.trello.com/1/cards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

### Passo 8 — Criar a checklist (somente se `checklist` foi informado)

Após o card ser criado com sucesso, use o `CARD_ID` retornado pela API para criar a checklist:

```bash
CARD_ID="<id retornado pela API no Passo 7>"
CHECKLIST_NAME="<valor do parâmetro checklist>"

# 1. Criar a checklist vinculada ao card
CHECKLIST_PAYLOAD=$(mktemp --suffix=.json)
cat > "$CHECKLIST_PAYLOAD" <<EOF
{"idCard": "${CARD_ID}", "name": "${CHECKLIST_NAME}"}
EOF

CHECKLIST_RESPONSE=$(curl -s -X POST "https://api.trello.com/1/checklists?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${CHECKLIST_PAYLOAD}")

rm -f "$CHECKLIST_PAYLOAD"
CHECKLIST_ID=$(echo "$CHECKLIST_RESPONSE" | grep -o '"id":"[^"]*"' | head -1 | cut -d'"' -f4)

# 2. Adicionar cada item à checklist (repetir para cada item inferido)
ITEM_PAYLOAD=$(mktemp --suffix=.json)
cat > "$ITEM_PAYLOAD" <<EOF
{"name": "<texto do item>"}
EOF

curl -s -X POST "https://api.trello.com/1/checklists/${CHECKLIST_ID}/checkItems?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${ITEM_PAYLOAD}"

rm -f "$ITEM_PAYLOAD"
```

Execute o curl de adição de item uma vez para cada item da lista inferida. Se a criação da checklist falhar, informe o erro mas não desfaça a criação do card.

### Passo 9 — Reportar o resultado

- **Sucesso:** exibir o nome do board, nome da coluna, ID e URL do card criado (campo `url` da resposta). Se uma checklist foi criada, informar o nome da checklist e a quantidade de itens adicionados.
- **Erro:** exibir a mensagem de erro retornada pela API.

---

## Fluxo de listagem de cards

Lista os cards de uma coluna de um board.

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `listar`) e extraia:

- `board`: valor do parâmetro `board:"..."` ou `board:...`
- `coluna`: valor do parâmetro `coluna:"..."` ou `coluna:...`

Se algum parâmetro não for informado, use `AskUserQuestion` para solicitá-lo.

### Passo 2 — Resolver board e coluna

Siga o mesmo procedimento do fluxo de criação (passos 2 e 3) para obter `BOARD_ID` e `LIST_ID`.

### Passo 3 — Listar os cards

```bash
curl -s "https://api.trello.com/1/lists/${LIST_ID}/cards?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&fields=name,url,labels,due,dateLastActivity"
```

### Passo 4 — Exibir os cards

Para cada card retornado, exibir:

```
- [<name>](<url>) — Labels: <labels[].name> | Vence: <due ou "sem prazo">
```

Se não houver cards, informar que a coluna está vazia.

---

## Fluxo de atualização

Atualiza o título e/ou a descrição de um card existente.

### Passo 1 — Extrair o ID e os novos valores

Analise `$ARGUMENTS` (após remover o prefixo `atualizar`). Extraia o ID ou URL do card (mesmo critério do fluxo de leitura). O restante do texto é a nova descrição/título a ser aplicado.

Use `AskUserQuestion` para perguntar o que deseja alterar: **Título**, **Descrição** ou **Ambos**.

### Passo 2 — Interpretar o novo conteúdo

- Se alterar o título: gere um título seguindo as mesmas regras do fluxo de criação (máximo 80 caracteres, sufixo ` (Criado pelo Claude Code)`).
- Se alterar a descrição: use os modelos em [card-models.md](references/card-models.md).

### Passo 3 — Confirmar com o usuário

Exiba os novos valores e use `AskUserQuestion` com as opções **Atualizar** e **Cancelar**.

### Passo 4 — Atualizar via API

```bash
CARD_ID="<id extraído>"
PAYLOAD_FILE=$(mktemp --suffix=.json)

cat > "$PAYLOAD_FILE" <<EOF
{
  "name": "<novo título, se alterado>",
  "desc": "<nova descrição, se alterada>"
}
EOF

curl -s -X PUT "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${PAYLOAD_FILE}"

rm -f "$PAYLOAD_FILE"
```

Inclua no JSON apenas os campos que foram alterados.

### Passo 5 — Reportar o resultado

Exibir o ID e a URL do card atualizado, informando o que foi alterado.

---

## Fluxo de movimentação

Move um card para outra coluna (e opcionalmente outro board).

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `mover`). Extraia:

- ID ou URL do card (mesmo critério do fluxo de leitura)
- `board`: nome do board de destino (opcional; se omitido, permanece no mesmo board)
- `coluna`: nome da coluna de destino

Se `coluna` não for informado, use `AskUserQuestion` para solicitá-lo.

### Passo 2 — Resolver o board e a coluna de destino

Se `board` foi informado, siga o passo 2 do fluxo de criação para obter o `BOARD_ID` de destino. Caso contrário, busque o board atual do card via:

```bash
curl -s "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}&fields=idBoard"
```

Em seguida, resolva a coluna de destino (passo 3 do fluxo de criação) para obter `TARGET_LIST_ID`.

### Passo 3 — Mover via API

```bash
curl -s -X PUT "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"idList\": \"${TARGET_LIST_ID}\"}"
```

### Passo 4 — Reportar o resultado

Informar o nome do board e da coluna de destino, o ID e a URL do card movido.

---

## Fluxo de arquivamento

Arquiva (fecha) um card existente.

### Passo 1 — Extrair o ID do card

Analise `$ARGUMENTS` (após remover o prefixo `arquivar`). Extraia o ID ou URL do card (mesmo critério do fluxo de leitura).

### Passo 2 — Confirmar com o usuário

Exiba o ID do card e use `AskUserQuestion` com as opções **Arquivar** e **Cancelar**. Se possível, busque primeiro o nome do card (passo 1 do fluxo de leitura) para exibi-lo na confirmação.

### Passo 3 — Arquivar via API

```bash
curl -s -X PUT "https://api.trello.com/1/cards/${CARD_ID}?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"closed": true}'
```

### Passo 4 — Reportar o resultado

Informar que o card foi arquivado com sucesso, exibindo o ID e o nome do card.

---

## Fluxo de listagem de checklists

Lista todos os checklists de um card, incluindo seus itens e status.

### Passo 1 — Extrair o ID do card

Analise `$ARGUMENTS` (após remover o prefixo `checklists`). Extraia o ID ou URL do card (mesmo critério do fluxo de leitura).

### Passo 2 — Buscar checklists via API

```bash
curl -s "https://api.trello.com/1/cards/${CARD_ID}/checklists?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}"
```

### Passo 3 — Exibir os checklists

Para cada checklist retornado, exibir nome, ID e seus itens:

```
Checklist: <name> (ID: <id>)
  [x] <item concluído>
  [ ] <item pendente>
```

Se não houver checklists, informar que o card não possui nenhum.

---

## Fluxo de adição de item a checklist existente

Adiciona um novo item a um checklist já existente em um card.

### Passo 1 — Extrair parâmetros

Analise `$ARGUMENTS` (após remover o prefixo `checklist-item`). O primeiro token é o `CHECKLIST_ID` e o restante é o texto do item.

Se o texto do item não for informado, use `AskUserQuestion` para solicitá-lo.

### Passo 2 — Adicionar o item via API

```bash
CHECKLIST_ID="<id extraído>"
ITEM_PAYLOAD=$(mktemp --suffix=.json)

cat > "$ITEM_PAYLOAD" <<EOF
{"name": "<texto do item>"}
EOF

curl -s -X POST "https://api.trello.com/1/checklists/${CHECKLIST_ID}/checkItems?key=${TRELLO_API_KEY}&token=${TRELLO_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  --data-binary "@${ITEM_PAYLOAD}"

rm -f "$ITEM_PAYLOAD"
```

### Passo 3 — Reportar o resultado

Informar que o item foi adicionado ao checklist, exibindo o nome do item e o ID do checklist.
