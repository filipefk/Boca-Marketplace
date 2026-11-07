---
name: google-sheets
description: Acessa e edita planilhas do Google Sheets via chamadas curl na API REST v4. Suporta listar abas, ler intervalos, escrever, anexar e limpar valores, criar/renomear/excluir abas, aplicar formatação (negrito, congelar painel, largura de coluna, validação de dados, formatação condicional, mesclagem) e criar planilhas novas, a partir de uma URL ou ID. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN com escopo de escrita (mesmas credenciais das skills google-drive, google-docs e google-slides — veja ../google-drive/references/oauth-setup.md).
---

## Como usar

**Leitura:**

```
/google-sheets abas <url ou ID da planilha>
/google-sheets ler <url ou ID da planilha> [intervalo]
```

**Escrita:**

```
/google-sheets escrever <url ou ID> <intervalo> <o que escrever>
/google-sheets anexar <url ou ID> <aba> <o que acrescentar>
/google-sheets limpar <url ou ID> <intervalo>
/google-sheets criar-aba <url ou ID> <nome da aba>
/google-sheets renomear-aba <url ou ID> <nome atual> <novo nome>
/google-sheets excluir-aba <url ou ID> <nome da aba>
/google-sheets formatar <url ou ID> <descrição do que formatar>
/google-sheets criar-planilha <título> [pasta do Drive]
```

Exemplos de `intervalo` (notação A1): `Vendas`, `Vendas!A1:D20`, `A1:C10` (usa a primeira aba).

O conteúdo a escrever pode vir em linguagem natural, como tabela markdown, como CSV colado ou do contexto da conversa — interprete e monte a matriz de valores (lista de linhas).

## Regras de segurança

**Sempre peça confirmação explícita ao usuário antes de:**

- sobrescrever células que já têm conteúdo;
- limpar um intervalo;
- excluir uma aba.

Ao pedir a confirmação, mostre o que será afetado: planilha, aba, intervalo, quantidade de linhas/células e um resumo do que existe hoje ali. Para descobrir isso, leia o intervalo de destino com o fluxo de leitura **antes** de escrever.

**Não exigem confirmação:** escrever em intervalo comprovadamente vazio, criar aba nova, criar planilha nova, formatar (formatação não destrói conteúdo).

Nunca exclua nem sobrescreva algo por iniciativa própria — só quando o usuário pedir.

## Referências

- [Receitas de formatação (`spreadsheets:batchUpdate`)](references/batchupdate-receitas.md) — payloads prontos para negrito, congelar painel, largura, validação de dados, formatação condicional, bordas e mesclagem.
- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](../google-drive/references/oauth-setup.md) — mesmo processo usado pela skill `google-drive`.

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0 (tipo "Aplicativo para computador")        |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md).

**Sobre o escopo:** os fluxos de **leitura** funcionam com `drive.readonly` ou `spreadsheets.readonly`. Os fluxos de **escrita** exigem o escopo completo `https://www.googleapis.com/auth/spreadsheets` (e, para mover uma planilha nova para uma pasta, também `https://www.googleapis.com/auth/drive`). Se uma chamada de escrita retornar `403` com `insufficientPermissions` ou `ACCESS_TOKEN_SCOPE_INSUFFICIENT`, oriente o usuário a regenerar o refresh token no OAuth Playground incluindo esses escopos.

## Ambiente

Armadilhas já observadas em execução real. Ler antes de escrever o primeiro comando — todas
custaram uma requisição perdida.

### Sem `jq` e sem `python`

Este ambiente **não tem `jq` nem `python` utilizável** (o `python` do PATH é o stub da Microsoft
Store, que não executa). Não escreva comandos que dependam deles.

Monte o JSON você mesmo e envie com `--data-binary @arquivo`. Isso evita todo problema de escape
de aspas e preserva acentuação UTF-8.

### Como gravar o arquivo JSON

Heredoc **entre aspas simples** (`<<'JSON'`) funciona para payloads pequenos, mas **falha em
payloads grandes ou com muitas linhas** — o bash retorna `unexpected EOF while looking for
matching` e o arquivo nem chega a ser criado. Aconteceu duas vezes com payloads de dezenas de
linhas contendo acentuação e trechos que parecem shell.

Para qualquer payload não trivial, **grave o arquivo com a ferramenta Write** e use o Bash apenas
para o `curl`. É mais confiável e o JSON fica legível.

### Acentos no nome da aba

Passar um intervalo acentuado direto na linha de comando **corrompe os bytes**:
`--data-urlencode "ranges=Página1!A1"` chega ao Google como `P gina1` e devolve
`400 Unable to parse range`.

Duas saídas, ambas verificadas:

1. **Gravar o intervalo num arquivo** e referenciá-lo: `--data-urlencode "ranges@$SP/range.txt"`
   (com `printf '%s'`, nunca `echo`, para não incluir o `\n`). É a forma geral, usada no fluxo de
   leitura.
2. **Percent-encoding manual** na URL: `?ranges=P%C3%A1gina1%21A1%3AC3`.

Os fluxos de escrita não sofrem disso porque usam apenas endpoints que recebem o intervalo
**no corpo** da requisição (`values:batchUpdate`, `values:batchClear`,
`spreadsheets:batchUpdate`). Prefira sempre esses.

### Nomes de função em português

Com `valueInputOption: USER_ENTERED`, a maioria das funções escritas em português é aceita e
convertida para o nome canônico ao ser gravada — `MÉDIA`→`AVERAGE`, `MÍNIMO`→`MIN`,
`ARRED`→`ROUND`, `SEERRO`→`IFERROR`, `ESCOLHER`→`CHOOSE`, `CORRESP`→`MATCH`, `ÍNDICE`→`INDEX`.

**Mas nem todas.** `UNIRTEXTO` e `FILTRO` são gravadas literalmente, sem conversão, e a fórmula
passa a dar erro — silenciosamente, se estiver dentro de um `SEERRO`, que devolve célula vazia
em vez de mensagem.

Não confie na conversão. **Escreva os nomes canônicos em inglês** (`TEXTJOIN`, `FILTER`,
`AVERAGE`, `MIN`…) — funcionam sempre, e o Google exibe ao usuário no idioma da planilha dele.

Depois de gravar qualquer fórmula, leia de volta com `valueRenderOption=FORMULA`: se o nome
voltar em português, ela não foi reconhecida.

O **separador de argumentos** segue a planilha: `;` em planilhas pt-BR, `,` em en-US. Confira
lendo uma fórmula existente antes de escrever as suas.

### Separador decimal

Números dentro de **campos de configuração** de `spreadsheets:batchUpdate` seguem a localidade da
planilha. Em planilha pt-BR, o ponto médio de um gradiente `"value": "2.5"` é rejeitado com
`Invalid InterpolationPoint.value`; o aceito é `"2,5"`.

Isso vale para os campos textuais de condição (`InterpolationPoint`, `BooleanCondition`), não
para os campos numéricos do JSON (`pixelSize`, `red`/`green`/`blue`), que são números de verdade
e usam ponto normalmente.

### `batchUpdate` é atômico

Se **uma** requisição do array falhar, **nenhuma** é aplicada. Um lote de 40 formatações morre
inteiro por causa de um `2.5`. Isso é bom (não deixa estado pela metade), mas significa que vale
conferir os detalhes sensíveis à localidade antes de mandar um lote grande — ou separar as
requisições arriscadas num lote próprio.

---

## Passo comum — Obter um access token

Antes de qualquer chamada à API do Sheets, troque o refresh token por um access token (válido por ~1 hora):

```bash
GOOGLE_CLIENT_ID="${GOOGLE_CLIENT_ID:?Variável GOOGLE_CLIENT_ID não encontrada}"
GOOGLE_CLIENT_SECRET="${GOOGLE_CLIENT_SECRET:?Variável GOOGLE_CLIENT_SECRET não encontrada}"
GOOGLE_REFRESH_TOKEN="${GOOGLE_REFRESH_TOKEN:?Variável GOOGLE_REFRESH_TOKEN não encontrada}"

TOKEN_RESPONSE=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -d client_id="${GOOGLE_CLIENT_ID}" \
  -d client_secret="${GOOGLE_CLIENT_SECRET}" \
  -d refresh_token="${GOOGLE_REFRESH_TOKEN}" \
  -d grant_type=refresh_token)

ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | grep -o '"access_token"[[:space:]]*:[[:space:]]*"[^"]*"' | cut -d'"' -f4)
```

Se `ACCESS_TOKEN` vier vazio, exiba o conteúdo de `$TOKEN_RESPONSE` como erro e encerre (provavelmente o refresh token expirou ou foi revogado — oriente o usuário a refazer o passo de autorização em [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md)).

O token vale para vários comandos: obtenha uma vez por bloco de trabalho, não a cada chamada. Como o shell não mantém estado entre invocações do Bash, agrupe o token e as chamadas que dependem dele **no mesmo comando**.

---

## Passo comum — Extrair o ID da planilha

Analise o argumento recebido (após remover o prefixo do comando). O primeiro token (até o próximo espaço) é a URL ou ID da planilha; o restante da string são os demais parâmetros do comando.

Se for uma URL do Sheets, extraia o ID:

- `https://docs.google.com/spreadsheets/d/<ID>/edit` → `<ID>`
- `https://docs.google.com/spreadsheets/d/<ID>/edit?usp=sharing` → `<ID>` (ignorar querystring)
- `https://docs.google.com/spreadsheets/d/<ID>/edit#gid=123456` → `<ID>` (ignorar o `#gid=...`)

Se não for uma URL, trate o token inteiro como o próprio ID da planilha.

Se nenhum argumento foi informado, use `AskUserQuestion` para solicitar a URL ou ID da planilha.

---

## Passo comum — Enviar um corpo JSON

Padrão usado por todos os fluxos de escrita. São dois passos separados:

**1. Gravar o payload.** Para qualquer coisa além de um punhado de linhas, use a ferramenta
**Write** para criar o arquivo `.json` no diretório de scratchpad da sessão. Heredoc só para
payloads curtos — ver "Como gravar o arquivo JSON" na seção Ambiente.

**2. Enviar:**

```bash
PAYLOAD="<diretório de scratchpad da sessão>/sheets-payload.json"

curl -s -X POST "<endpoint>" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=UTF-8" \
  --data-binary @"$PAYLOAD"
```

Se a resposta contiver um campo `error`, exiba `error.message` e encerre sem tentar novamente com os mesmos parâmetros.

**Token entre comandos:** o shell não mantém variáveis entre invocações do Bash. Grave o access
token num arquivo (`echo "$ACCESS_TOKEN" > "$SP/at.txt"`) e releia com `AT=$(cat "$SP/at.txt")`
nos comandos seguintes, em vez de renovar o token a cada chamada.

**Limite de tamanho:** a API aceita requisições grandes, mas payloads muito extensos ficam difíceis de depurar. Acima de ~500 linhas de dados, divida em lotes e faça uma chamada por lote.

**Conferir depois de escrever:** leia de volta o que gravou. Vale sobretudo para fórmulas
(`valueRenderOption=FORMULA`) e para a contagem de linhas — errar em uma linha o índice de
destino é fácil e passa despercebido, porque a API grava com sucesso no lugar errado.

---

## Passo comum — Descobrir o `sheetId` numérico

Os fluxos de aba e de formatação usam o `sheetId` numérico, não o nome da aba. Obtenha-o com o fluxo de listagem de abas (abaixo) e case pelo `title`.

O `sheetId` da primeira aba de uma planilha nova costuma ser `0`, mas **não assuma isso** — sempre consulte.

---

## Fluxo de listagem de abas

### Passo 1 — Extrair o ID da planilha

Ver "Passo comum — Extrair o ID da planilha" acima.

### Passo 2 — Obter o access token

Executar o "Passo comum — Obter um access token" acima.

### Passo 3 — Buscar as abas

```bash
SPREADSHEET_ID="<id extraído no passo 1>"

curl -s -G "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}" \
  --data-urlencode "fields=properties(title),sheets(properties(sheetId,title,gridProperties))" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Passo 4 — Exibir o resultado

Se a resposta contiver um campo `error`, exiba a mensagem de erro (ex: planilha não encontrada, sem permissão de acesso) e encerre.

Caso contrário, exiba o título da planilha (`properties.title`) e, para cada item em `sheets`, exibir:

```
- <properties.title> (id <sheetId>) — <gridProperties.rowCount> linhas x <gridProperties.columnCount> colunas
```

---

## Fluxo de leitura de intervalo

### Passo 1 — Extrair o ID da planilha e o intervalo

Ver "Passo comum — Extrair o ID da planilha" acima para o ID.

Se um intervalo foi informado como segundo token do argumento, use-o diretamente (notação A1, ex: `Vendas!A1:D20` ou apenas `Vendas`).

Se nenhum intervalo foi informado, execute o "Passo 3" do fluxo de listagem de abas para descobrir o nome da primeira aba e use esse nome como intervalo (retorna todos os dados preenchidos daquela aba).

### Passo 2 — Obter o access token

Executar o "Passo comum — Obter um access token" acima (pode ser reaproveitado se já obtido no passo anterior).

### Passo 3 — Ler os valores

O intervalo **precisa** ir para o curl através de um arquivo, e não direto na linha de comando —
ver "Acentos no nome da aba" na seção Ambiente. Passar `--data-urlencode "ranges=${RANGE}"` com
um nome acentuado devolve `400 Unable to parse range`.

```bash
SPREADSHEET_ID="<id extraído no passo 1>"

printf '%s' '<intervalo definido no passo 1>' > "$SP/range.txt"

curl -s -G "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchGet" \
  --data-urlencode "ranges@$SP/range.txt" \
  --data-urlencode "majorDimension=ROWS" \
  --data-urlencode "valueRenderOption=FORMATTED_VALUE" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

Use `printf '%s'` e não `echo`: o `\n` que o `echo` acrescenta entra no valor codificado e
quebra o intervalo.

Para inspecionar **fórmulas** em vez do resultado calculado, troque `valueRenderOption` por
`FORMULA`. Vale sempre que você acabou de gravar uma fórmula e quer confirmar que ela foi
aceita — ver "Nomes de função em português" na seção Ambiente.

### Passo 4 — Exibir o resultado

Se a resposta contiver um campo `error`, exiba a mensagem de erro (ex: intervalo inválido, planilha não encontrada, sem permissão de acesso) e encerre.

Caso contrário, pegue `valueRanges[0].values` (lista de linhas, cada uma uma lista de células) e monte uma tabela markdown com o conteúdo:

- Se a primeira linha parecer um cabeçalho (valores distintos, texto curto), use-a como cabeçalho da tabela.
- Linhas mais curtas que as demais têm células vazias implícitas no final — preencha com vazio ao montar a tabela.

Se `values` estiver ausente ou vazio, informe que o intervalo não possui dados.

---

## Fluxo de escrita de valores

### Passo 1 — Extrair ID, intervalo e conteúdo

Ver "Passo comum — Extrair o ID da planilha". O segundo token é o intervalo de destino (notação A1); o restante é o conteúdo a escrever.

Basta indicar a **célula superior esquerda** (ex: `Avaliação!A1`) — a API expande conforme o tamanho da matriz enviada. Informar o intervalo completo só é necessário quando você quiser garantir que nada além dele seja tocado.

Converta o conteúdo para uma matriz de linhas. Cada linha é uma lista de células; use `""` para célula vazia e `null` para "não alterar esta célula".

### Passo 2 — Verificar o destino e confirmar

Execute o fluxo de leitura no intervalo de destino. Se houver dados, **pare e peça confirmação** ao usuário (ver "Regras de segurança"), mostrando quantas linhas serão sobrescritas e o que há ali hoje. Se estiver vazio, siga direto.

### Passo 3 — Obter o access token

Executar o "Passo comum — Obter um access token".

### Passo 4 — Escrever

Grave o payload abaixo com a ferramenta **Write** (o heredoc mostrado serve só para ilustrar o
formato — em payload real ele quebra; ver seção Ambiente) e envie com o `curl` do final.

```bash
SPREADSHEET_ID="<id>"
PAYLOAD="<scratchpad>/sheets-payload.json"

cat > "$PAYLOAD" <<'JSON'
{
  "valueInputOption": "USER_ENTERED",
  "data": [
    {
      "range": "Avaliação!A1",
      "majorDimension": "ROWS",
      "values": [
        ["Dimensão", "Nível", "Observação"],
        ["Versionamento", 3, "MRs obrigatórios"],
        ["Testes", 1, ""]
      ]
    }
  ]
}
JSON

curl -s -X POST "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchUpdate" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=UTF-8" \
  --data-binary @"$PAYLOAD"
```

Sobre `valueInputOption`:

| Valor | Comportamento | Quando usar |
|---|---|---|
| `USER_ENTERED` | Interpreta como se o usuário tivesse digitado: fórmulas (`=SOMA(...)`) viram fórmulas, `10%` vira percentual, datas viram datas | Padrão. Use sempre que quiser fórmulas ou tipagem automática |
| `RAW` | Grava literalmente como texto/número, sem interpretar | Códigos com zero à esquerda, textos que começam com `=`, `+` ou `-`, IDs |

O array `data` aceita **vários intervalos na mesma chamada** — use isso para escrever em abas diferentes de uma só vez, em vez de fazer várias requisições.

### Passo 5 — Confirmar

A resposta traz `totalUpdatedCells`, `totalUpdatedRows` e `totalUpdatedColumns`. Informe ao usuário quantas células foram gravadas e em qual intervalo, com o link da planilha.

---

## Fluxo de anexar linhas

Acrescenta linhas **após** a última linha preenchida, sem sobrescrever nada.

Como o endpoint `values:append` exige o intervalo na URL (o que quebra com nomes de aba acentuados neste ambiente), faça em dois passos:

1. **Descubra a última linha preenchida:** execute o fluxo de leitura na aba inteira (ex: `ranges=Avaliação`) e conte as linhas de `valueRanges[0].values`. Se vierem 42 linhas, a próxima livre é a 43.
2. **Escreva:** execute o fluxo de escrita com `range` apontando para essa linha (ex: `Avaliação!A43`).

Anexar não sobrescreve, então **não exige confirmação** — desde que você tenha realmente conferido a última linha no passo 1.

---

## Fluxo de limpeza de intervalo

Remove os **valores** de um intervalo, preservando a formatação.

### Passo 1 — Confirmar

Leia o intervalo primeiro e mostre ao usuário o que será apagado. **Exige confirmação explícita.**

### Passo 2 — Limpar

```bash
SPREADSHEET_ID="<id>"
PAYLOAD="<scratchpad>/sheets-payload.json"

cat > "$PAYLOAD" <<'JSON'
{
  "ranges": ["Avaliação!A2:F200"]
}
JSON

curl -s -X POST "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchClear" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=UTF-8" \
  --data-binary @"$PAYLOAD"
```

Para apagar **também a formatação**, use `spreadsheets:batchUpdate` com uma requisição `updateCells` — ver [receitas de formatação](references/batchupdate-receitas.md).

---

## Fluxo de criar aba

```bash
SPREADSHEET_ID="<id>"
PAYLOAD="<scratchpad>/sheets-payload.json"

cat > "$PAYLOAD" <<'JSON'
{
  "requests": [
    {
      "addSheet": {
        "properties": {
          "title": "Avaliação",
          "index": 1,
          "gridProperties": { "rowCount": 300, "columnCount": 12, "frozenRowCount": 1 }
        }
      }
    }
  ]
}
JSON

curl -s -X POST "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}:batchUpdate" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=UTF-8" \
  --data-binary @"$PAYLOAD"
```

A resposta traz o `sheetId` da aba criada em `replies[0].addSheet.properties.sheetId` — guarde-o para as chamadas de formatação seguintes.

Se a aba já existir, a API retorna erro. Liste as abas antes para decidir entre criar ou reaproveitar.

Várias abas podem ser criadas em uma única chamada, colocando vários `addSheet` no array `requests`.

---

## Fluxo de renomear aba

Precisa do `sheetId` numérico (ver "Passo comum — Descobrir o `sheetId` numérico").

```bash
cat > "$PAYLOAD" <<'JSON'
{
  "requests": [
    {
      "updateSheetProperties": {
        "properties": { "sheetId": 0, "title": "Resultado" },
        "fields": "title"
      }
    }
  ]
}
JSON
```

Envie para `.../spreadsheets/${SPREADSHEET_ID}:batchUpdate` como nos exemplos acima.

O campo `fields` é obrigatório e lista quais propriedades serão alteradas — sem ele nada muda, e propriedades não listadas ficam intactas.

---

## Fluxo de excluir aba

**Exige confirmação explícita.** A exclusão apaga todo o conteúdo da aba e não é desfeita pela API.

Antes de perguntar, leia a aba e informe ao usuário quantas linhas de dados serão perdidas.

```bash
cat > "$PAYLOAD" <<'JSON'
{
  "requests": [ { "deleteSheet": { "sheetId": 123456789 } } ]
}
JSON
```

Uma planilha não pode ficar sem nenhuma aba — a API rejeita a exclusão da última.

---

## Fluxo de formatação

Interprete o pedido em linguagem natural e monte as requisições correspondentes em `spreadsheets:batchUpdate`. Os payloads prontos estão em [references/batchupdate-receitas.md](references/batchupdate-receitas.md), que cobre:

- cabeçalho em negrito com fundo colorido
- congelar linhas/colunas
- largura de coluna e ajuste automático
- quebra de texto e alinhamento
- formato numérico (percentual, moeda, data)
- validação de dados (lista suspensa)
- formatação condicional (escala de cores e por regra)
- bordas, mesclagem e faixas alternadas
- limpar formatação

Agrupe todas as requisições no mesmo array `requests` de uma única chamada — a API as aplica em ordem, de forma atômica, e isso é bem mais rápido que uma chamada por ajuste.

Formatação não destrói conteúdo, então **não exige confirmação**.

---

## Fluxo de criar planilha

### Passo 1 — Criar

```bash
PAYLOAD="<scratchpad>/sheets-payload.json"

cat > "$PAYLOAD" <<'JSON'
{
  "properties": { "title": "Scorecard Maturidade Esteira", "locale": "pt_BR" },
  "sheets": [
    { "properties": { "title": "Instruções" } },
    { "properties": { "title": "Avaliação", "gridProperties": { "frozenRowCount": 1 } } }
  ]
}
JSON

curl -s -X POST "https://sheets.googleapis.com/v4/spreadsheets" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=UTF-8" \
  --data-binary @"$PAYLOAD"
```

A resposta traz `spreadsheetId` e `spreadsheetUrl` — mostre a URL ao usuário.

### Passo 2 — Mover para uma pasta do Drive (opcional)

A planilha nasce na raiz do Drive. Para movê-la, é preciso o escopo `https://www.googleapis.com/auth/drive`:

```bash
NEW_ID="<spreadsheetId da resposta>"
FOLDER_ID="<id da pasta de destino>"

ROOT_ID=$(curl -s "https://www.googleapis.com/drive/v3/files/${NEW_ID}?fields=parents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" | grep -o '"[^"]*"' | tail -1 | tr -d '"')

curl -s -X PATCH "https://www.googleapis.com/drive/v3/files/${NEW_ID}?addParents=${FOLDER_ID}&removeParents=${ROOT_ID}&fields=id,parents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

O ID da pasta sai da URL do Drive: `https://drive.google.com/drive/folders/<FOLDER_ID>`.

---

## Erros comuns

| Erro | Causa provável | O que fazer |
|---|---|---|
| `403` `ACCESS_TOKEN_SCOPE_INSUFFICIENT` | Refresh token só tem escopo de leitura | Regerar o token com o escopo `spreadsheets` completo |
| `403` `The caller does not have permission` | A conta do token não tem acesso à planilha | Pedir ao usuário que compartilhe a planilha com essa conta |
| `404` | ID errado, ou a URL era de um Doc/Slide | Conferir o ID extraído da URL |
| `400` `Unable to parse range` **com letra acentuada faltando** (ex: `P gina1`) | O intervalo foi passado direto na linha de comando e os bytes se perderam | Usar `--data-urlencode "ranges@arquivo"` — ver "Acentos no nome da aba" |
| `400` `Unable to parse range` sem corrupção | Nome de aba com espaço sem aspas simples, ou aba inexistente | Usar `'Nome da Aba'!A1`; listar as abas para conferir o nome exato |
| `400` `Invalid requests[N]` | Falta o campo `fields` num `update*`, ou `sheetId` inexistente | Incluir `fields`; reconsultar o `sheetId`. O `N` diz qual requisição do array falhou |
| `400` `Invalid InterpolationPoint.value` | Separador decimal errado para a localidade da planilha | Trocar `2.5` por `2,5` em planilha pt-BR |
| Nada foi aplicado, mas só uma requisição estava errada | `batchUpdate` é atômico | Corrigir a requisição apontada e **reenviar o lote inteiro** |
| Escreveu, mas as fórmulas viraram texto | `valueInputOption` estava como `RAW` | Reenviar com `USER_ENTERED` |
| Fórmula gravou sem erro e a célula ficou vazia | Nome de função não reconhecido (ex: `UNIRTEXTO`, `FILTRO`) dentro de um `SEERRO`, que engole o erro | Ler com `valueRenderOption=FORMULA`; reescrever com o nome canônico em inglês |
| Gravou no lugar errado, sem erro | Índice de linha calculado errado ao montar o array de valores | Ler de volta o intervalo e conferir contra o layout pretendido antes de seguir |
| `429` | Excesso de requisições (limite por minuto por usuário) | Agrupar tudo em um único `batchUpdate` e reduzir o número de chamadas |
