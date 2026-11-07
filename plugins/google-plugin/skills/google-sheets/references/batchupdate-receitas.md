# Receitas de `spreadsheets:batchUpdate`

Payloads prontos para as formatações mais comuns. Todos vão para o mesmo endpoint:

```
POST https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}:batchUpdate
```

Envie conforme o "Passo comum — Enviar um corpo JSON" da [SKILL.md](../SKILL.md).

## Como combinar

O array `requests` aceita quantas requisições você quiser, aplicadas **em ordem** e de forma atômica (se uma falhar, nenhuma é aplicada). Sempre agrupe tudo numa chamada só:

```json
{
  "requests": [
    { "updateSheetProperties": { "...": "..." } },
    { "repeatCell": { "...": "..." } },
    { "setDataValidation": { "...": "..." } }
  ]
}
```

## Convenções

**`GridRange`** — o objeto que delimita as células. Índices começam em **0** e o fim é **exclusivo**:

```json
{ "sheetId": 0, "startRowIndex": 0, "endRowIndex": 1, "startColumnIndex": 0, "endColumnIndex": 6 }
```

Isso é a linha 1, colunas A a F. Omitir um limite significa "até o fim": sem `endRowIndex`, vale até a última linha da aba.

**`fields`** — obrigatório em toda requisição `update*` e `repeatCell`. Lista, separada por vírgula, das propriedades que serão alteradas; o que não está listado fica intacto. Use `"*"` só quando quiser mesmo sobrescrever tudo. Aceita notação de caminho: `userEnteredFormat(backgroundColor,textFormat)`.

**Cores** — RGB em ponto flutuante de 0 a 1, não 0–255. `#4A86E8` vira `{"red":0.29,"green":0.53,"blue":0.91}`.

---

## Cabeçalho em negrito com fundo colorido

```json
{
  "requests": [
    {
      "repeatCell": {
        "range": { "sheetId": 0, "startRowIndex": 0, "endRowIndex": 1 },
        "cell": {
          "userEnteredFormat": {
            "backgroundColor": { "red": 0.24, "green": 0.35, "blue": 0.5 },
            "horizontalAlignment": "CENTER",
            "verticalAlignment": "MIDDLE",
            "wrapStrategy": "WRAP",
            "textFormat": {
              "foregroundColor": { "red": 1, "green": 1, "blue": 1 },
              "fontSize": 11,
              "bold": true
            }
          }
        },
        "fields": "userEnteredFormat(backgroundColor,horizontalAlignment,verticalAlignment,wrapStrategy,textFormat)"
      }
    }
  ]
}
```

## Congelar linhas e colunas

```json
{
  "requests": [
    {
      "updateSheetProperties": {
        "properties": { "sheetId": 0, "gridProperties": { "frozenRowCount": 1, "frozenColumnCount": 2 } },
        "fields": "gridProperties(frozenRowCount,frozenColumnCount)"
      }
    }
  ]
}
```

## Largura de coluna

Largura fixa em pixels (colunas A–C):

```json
{
  "requests": [
    {
      "updateDimensionProperties": {
        "range": { "sheetId": 0, "dimension": "COLUMNS", "startIndex": 0, "endIndex": 3 },
        "properties": { "pixelSize": 220 },
        "fields": "pixelSize"
      }
    }
  ]
}
```

Ajuste automático ao conteúdo:

```json
{
  "requests": [
    {
      "autoResizeDimensions": {
        "dimensions": { "sheetId": 0, "dimension": "COLUMNS", "startIndex": 0, "endIndex": 8 }
      }
    }
  ]
}
```

`autoResizeDimensions` roda sobre o conteúdo que já está na planilha — aplique **depois** de escrever os valores, nunca antes.

## Quebra de texto e alinhamento

```json
{
  "requests": [
    {
      "repeatCell": {
        "range": { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 2, "endColumnIndex": 4 },
        "cell": {
          "userEnteredFormat": { "wrapStrategy": "WRAP", "verticalAlignment": "TOP" }
        },
        "fields": "userEnteredFormat(wrapStrategy,verticalAlignment)"
      }
    }
  ]
}
```

`wrapStrategy`: `WRAP` (quebra e aumenta a linha), `CLIP` (corta), `OVERFLOW_CELL` (transborda para a célula vizinha se estiver vazia).

## Formato numérico

```json
{
  "requests": [
    {
      "repeatCell": {
        "range": { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 4, "endColumnIndex": 5 },
        "cell": {
          "userEnteredFormat": { "numberFormat": { "type": "PERCENT", "pattern": "0.0%" } }
        },
        "fields": "userEnteredFormat.numberFormat"
      }
    }
  ]
}
```

Padrões úteis:

| Tipo | `pattern` | Resultado |
|---|---|---|
| `NUMBER` | `#,##0.0` | `1.234,5` |
| `PERCENT` | `0.0%` | `82,5%` |
| `CURRENCY` | `"R$" #,##0.00` | `R$ 1.234,00` |
| `DATE` | `dd/mm/yyyy` | `01/09/2026` |
| `DATE_TIME` | `dd/mm/yyyy hh:mm` | `01/09/2026 14:30` |

## Validação de dados (lista suspensa)

```json
{
  "requests": [
    {
      "setDataValidation": {
        "range": { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 4, "endColumnIndex": 5 },
        "rule": {
          "condition": {
            "type": "ONE_OF_LIST",
            "values": [
              { "userEnteredValue": "Sim" },
              { "userEnteredValue": "Não" },
              { "userEnteredValue": "N/A" }
            ]
          },
          "showCustomUi": true,
          "strict": true
        }
      }
    }
  ]
}
```

`showCustomUi: true` desenha o seletor suspenso; `strict: true` rejeita valores fora da lista (com `false`, apenas avisa).

Para uma lista vinda de outro intervalo, troque a condição por:

```json
{ "type": "ONE_OF_RANGE", "values": [ { "userEnteredValue": "=Listas!$A$1:$A$10" } ] }
```

Para restringir a números de 0 a 5:

```json
{ "type": "NUMBER_BETWEEN", "values": [ { "userEnteredValue": "0" }, { "userEnteredValue": "5" } ] }
```

Para **remover** a validação, envie `setDataValidation` com o `range` e **sem** o campo `rule`.

## Formatação condicional — escala de cores

```json
{
  "requests": [
    {
      "addConditionalFormatRule": {
        "index": 0,
        "rule": {
          "ranges": [ { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 1, "endColumnIndex": 2 } ],
          "gradientRule": {
            "minpoint": { "color": { "red": 0.96, "green": 0.42, "blue": 0.42 }, "type": "NUMBER", "value": "0" },
            "midpoint": { "color": { "red": 1, "green": 0.85, "blue": 0.4 }, "type": "NUMBER", "value": "2,5" },
            "maxpoint": { "color": { "red": 0.42, "green": 0.78, "blue": 0.5 }, "type": "NUMBER", "value": "5" }
          }
        }
      }
    }
  ]
}
```

O `value` de um `InterpolationPoint` é **texto**, e segue o separador decimal da localidade da
planilha: `"2,5"` em planilha pt-BR, `"2.5"` em en-US. Errar isso devolve
`400 Invalid InterpolationPoint.value` e, como o `batchUpdate` é atômico, derruba o lote inteiro.

## Formatação condicional — por regra

Pinta a linha inteira quando a coluna E for "Não":

```json
{
  "requests": [
    {
      "addConditionalFormatRule": {
        "index": 0,
        "rule": {
          "ranges": [ { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 0, "endColumnIndex": 7 } ],
          "booleanRule": {
            "condition": {
              "type": "CUSTOM_FORMULA",
              "values": [ { "userEnteredValue": "=$E2=\"Não\"" } ]
            },
            "format": {
              "backgroundColor": { "red": 0.99, "green": 0.9, "blue": 0.9 }
            }
          }
        }
      }
    }
  ]
}
```

Na fórmula personalizada, as referências são relativas à **primeira célula do range** — por isso `$E2` (coluna travada, linha livre) quando o range começa na linha 2.

Outros `condition.type` úteis: `TEXT_EQ`, `TEXT_CONTAINS`, `NUMBER_GREATER`, `NUMBER_LESS`, `BLANK`, `NOT_BLANK`.

`index: 0` coloca a regra no topo da pilha; regras anteriores continuam existindo. Para substituir uma regra, use `deleteConditionalFormatRule` com o índice dela antes de adicionar a nova.

## Bordas

```json
{
  "requests": [
    {
      "updateBorders": {
        "range": { "sheetId": 0, "startRowIndex": 0, "endRowIndex": 20, "startColumnIndex": 0, "endColumnIndex": 7 },
        "innerHorizontal": { "style": "SOLID", "width": 1, "color": { "red": 0.85, "green": 0.85, "blue": 0.85 } },
        "bottom": { "style": "SOLID_MEDIUM", "width": 2, "color": { "red": 0.4, "green": 0.4, "blue": 0.4 } }
      }
    }
  ]
}
```

Campos: `top`, `bottom`, `left`, `right`, `innerHorizontal`, `innerVertical`. Estilos: `SOLID`, `SOLID_MEDIUM`, `SOLID_THICK`, `DASHED`, `DOTTED`, `DOUBLE`, `NONE`.

## Faixas alternadas (zebrado)

```json
{
  "requests": [
    {
      "addBanding": {
        "bandedRange": {
          "range": { "sheetId": 0, "startRowIndex": 0, "endRowIndex": 100, "startColumnIndex": 0, "endColumnIndex": 7 },
          "rowProperties": {
            "headerColor": { "red": 0.24, "green": 0.35, "blue": 0.5 },
            "firstBandColor": { "red": 1, "green": 1, "blue": 1 },
            "secondBandColor": { "red": 0.95, "green": 0.96, "blue": 0.98 }
          }
        }
      }
    }
  ]
}
```

Faixas alternadas não podem se sobrepor a outra faixa já existente no mesmo intervalo.

## Mesclar células

```json
{
  "requests": [
    {
      "mergeCells": {
        "range": { "sheetId": 0, "startRowIndex": 0, "endRowIndex": 1, "startColumnIndex": 0, "endColumnIndex": 4 },
        "mergeType": "MERGE_ALL"
      }
    }
  ]
}
```

`mergeType`: `MERGE_ALL` (tudo numa célula), `MERGE_COLUMNS` (mescla por coluna), `MERGE_ROWS` (mescla por linha). Para desfazer, use `unmergeCells` com o mesmo `range`.

## Proteger um intervalo

Impede edição manual de células com fórmula, por exemplo:

```json
{
  "requests": [
    {
      "addProtectedRange": {
        "protectedRange": {
          "range": { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 1, "endColumnIndex": 3 },
          "description": "Colunas calculadas — não editar",
          "warningOnly": true
        }
      }
    }
  ]
}
```

`warningOnly: true` apenas avisa ao editar. Com `false`, só os editores listados em `editors` conseguem alterar.

## Limpar formatação (mantendo os valores)

```json
{
  "requests": [
    {
      "repeatCell": {
        "range": { "sheetId": 0 },
        "cell": {},
        "fields": "userEnteredFormat"
      }
    }
  ]
}
```

## Limpar valores E formatação

```json
{
  "requests": [
    {
      "updateCells": {
        "range": { "sheetId": 0, "startRowIndex": 1 },
        "fields": "*"
      }
    }
  ]
}
```

Destrutivo — **exige confirmação do usuário** antes de executar.

## Inserir e excluir linhas/colunas

```json
{
  "requests": [
    {
      "insertDimension": {
        "range": { "sheetId": 0, "dimension": "ROWS", "startIndex": 5, "endIndex": 8 },
        "inheritFromBefore": true
      }
    },
    {
      "deleteDimension": {
        "range": { "sheetId": 0, "dimension": "COLUMNS", "startIndex": 9, "endIndex": 26 }
      }
    }
  ]
}
```

`deleteDimension` apaga os dados junto — **exige confirmação**. Útil, sem risco, para remover as colunas vazias de sobra (J em diante) de uma aba nova.

## Ordenar e filtrar

Ordenar por uma coluna:

```json
{
  "requests": [
    {
      "sortRange": {
        "range": { "sheetId": 0, "startRowIndex": 1, "startColumnIndex": 0, "endColumnIndex": 7 },
        "sortSpecs": [ { "dimensionIndex": 1, "sortOrder": "ASCENDING" } ]
      }
    }
  ]
}
```

Criar o filtro básico da aba:

```json
{
  "requests": [
    { "setBasicFilter": { "filter": { "range": { "sheetId": 0, "startRowIndex": 0, "endColumnIndex": 7 } } } }
  ]
}
```

## Referência oficial

Lista completa de requisições e campos: <https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets/request>
