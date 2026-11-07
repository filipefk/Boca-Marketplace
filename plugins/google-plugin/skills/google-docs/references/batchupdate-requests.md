# Catálogo de requests do `documents.batchUpdate`

Endpoint: `POST https://docs.googleapis.com/v1/documents/{documentId}:batchUpdate`
Corpo: `{"requests": [ <request>, <request>, ... ]}`

Todas as requests do lote são aplicadas em ordem e de forma atômica: se uma falhar, nenhuma é aplicada.

## Como os índices funcionam

Diferente do Slides (que identifica elementos por `objectId`), o Docs identifica **posições no texto** por índice de caractere. Isso muda a forma de montar um lote:

- O corpo do documento começa no índice **1** (o índice 0 é reservado e não é gravável).
- `endIndex` de um parágrafo **inclui o `\n` final** que fecha o parágrafo. Apagar um intervalo até esse `endIndex` funde o parágrafo com o seguinte — para remover só o conteúdo, use `endIndex - 1`.
- **Cada request aplicada desloca todos os índices posteriores do mesmo lote.** Se você inserir 20 caracteres no índice 50, tudo que estava depois do 50 se move 20 posições para frente — mas as requests seguintes do mesmo lote continuam se referindo às posições que você calculou antes de qualquer inserção.

  Duas formas de lidar com isso dentro de um único lote:
  1. **Ordem decrescente de índice** — monte as requests de inserção da posição mais alta para a mais baixa. Assim, cada inserção só afeta índices que as requests seguintes (mais baixas) não usam.
  2. **`endOfSegmentLocation`** — em vez de `location.index`, use `endOfSegmentLocation` para anexar ao fim do corpo (ou de um cabeçalho/rodapé/nota). Imune a deslocamento, porque a posição é sempre recalculada como "o fim, agora".
- **Depois que uma tabela é inserida**, os índices do restante do documento mudam de um jeito difícil de prever no cliente (cada célula abre e fecha seu próprio segmento de índices). Releia a estrutura (`documents.get`) antes de montar o próximo lote em vez de tentar prever.
- `replaceAllText` **não depende de índice** — ele busca o texto em todo o documento (ou nos segmentos indicados) no momento da execução. É por isso que é o padrão preferido para preencher templates: não sofre com deslocamento.

---

## Padrão A — Substituição (documento com marcadores)

Use quando o documento template já tem estilos, cabeçalho/rodapé e estrutura prontos, com marcadores de texto tipo `{{cliente}}`.

### `replaceAllText`

Troca todas as ocorrências de um texto no documento inteiro, incluindo cabeçalho, rodapé e notas de rodapé.

```json
{
  "replaceAllText": {
    "containsText": { "text": "{{cliente}}", "matchCase": true },
    "replaceText": "ACME Ltda."
  }
}
```

Várias substituições podem ir no mesmo lote, em qualquer ordem — não há problema de deslocamento porque a busca é por texto, não por índice.

### `replaceImage`

Troca uma imagem existente por outra, mantendo posição e tamanho. `imageObjectId` vem da estrutura (campo `inlineObjectId` de um `InlineObjectElement`, não incluído na máscara `fields` padrão — peça `body(content(paragraph(elements(inlineObjectElement(inlineObjectId)))))` se precisar).

```json
{
  "replaceImage": {
    "imageObjectId": "kix.abc123",
    "uri": "https://exemplo.com/logo-novo.png",
    "imageReplaceMethod": "CENTER_CROP"
  }
}
```

A URL precisa ser publicamente acessível.

---

## Padrão B — Inserção posicional (documento em branco ou sem marcadores)

### `insertText`

```json
{
  "insertText": {
    "location": { "index": 1 },
    "text": "Relatório mensal\n"
  }
}
```

Para anexar sempre no fim, evitando cálculo de índice:

```json
{
  "insertText": {
    "endOfSegmentLocation": {},
    "text": "Mais uma linha ao final.\n"
  }
}
```

Cada `\n` inicia um novo parágrafo — é assim que se cria a separação entre título, corpo e itens de lista.

### `deleteContentRange`

```json
{
  "deleteContentRange": {
    "range": { "startIndex": 49, "endIndex": 209 }
  }
}
```

Lembrar: para apagar um parágrafo inteiro sem fundir com o próximo, use `endIndex - 1` do parágrafo (deixando o `\n` final intacto), a menos que a intenção seja remover a quebra também.

### `insertPageBreak`

```json
{
  "insertPageBreak": {
    "location": { "index": 350 }
  }
}
```

---

## Estilos de parágrafo e texto

### `updateParagraphStyle`

`namedStyleType` mais comuns: `TITLE`, `SUBTITLE`, `HEADING_1` a `HEADING_6`, `NORMAL_TEXT`. `fields` é obrigatório e indica quais propriedades aplicar.

```json
{
  "updateParagraphStyle": {
    "range": { "startIndex": 1, "endIndex": 18 },
    "paragraphStyle": { "namedStyleType": "HEADING_1" },
    "fields": "namedStyleType"
  }
}
```

### `updateTextStyle`

```json
{
  "updateTextStyle": {
    "range": { "startIndex": 1, "endIndex": 18 },
    "textStyle": {
      "bold": true,
      "fontSize": { "magnitude": 20, "unit": "PT" },
      "foregroundColor": { "color": { "rgbColor": { "red": 0.1, "green": 0.1, "blue": 0.1 } } }
    },
    "fields": "bold,fontSize,foregroundColor"
  }
}
```

Para um link:

```json
{
  "updateTextStyle": {
    "range": { "startIndex": 40, "endIndex": 55 },
    "textStyle": { "link": { "url": "https://exemplo.com" } },
    "fields": "link"
  }
}
```

### `createParagraphBullets` / `deleteParagraphBullets`

Presets mais usados: `BULLET_DISC_CIRCLE_SQUARE` (marcadores) e `NUMBERED_DECIMAL_ALPHA_ROMAN` (numerada).

```json
{
  "createParagraphBullets": {
    "range": { "startIndex": 60, "endIndex": 140 },
    "bulletPreset": "BULLET_DISC_CIRCLE_SQUARE"
  }
}
```

```json
{ "deleteParagraphBullets": { "range": { "startIndex": 60, "endIndex": 140 } } }
```

Evite formatar manualmente quando o template já define o estilo via `namedStyleType` — o ganho do template é justamente não precisar disso.

---

## Tabelas

### `insertTable`

```json
{
  "insertTable": {
    "location": { "index": 210 },
    "rows": 3,
    "columns": 2
  }
}
```

Depois de inserida, **releia a estrutura** — os `startIndex` de cada célula (`tableRows[].tableCells[].startIndex`) são necessários para escrever nelas e não dá para prever sem reler.

### Escrever em uma célula

Cada célula é, por dentro, um parágrafo próprio — insira texto no `startIndex` da célula igual a um `insertText` normal:

```json
{ "insertText": { "location": { "index": 214 }, "text": "Item" } }
```

### `insertTableRow` / `insertTableColumn` / `deleteTableRow`

```json
{
  "insertTableRow": {
    "tableCellLocation": { "tableStartLocation": { "index": 210 }, "rowIndex": 0, "columnIndex": 0 },
    "insertBelow": true
  }
}
```

```json
{ "deleteTableRow": { "tableCellLocation": { "tableStartLocation": { "index": 210 }, "rowIndex": 2, "columnIndex": 0 } } }
```

---

## Imagens

### `insertInlineImage`

A URL precisa ser publicamente acessível (o Google baixa a imagem no momento da chamada), em PNG/JPEG/GIF e até 50 MB. Links do Drive que exigem login **não funcionam**.

```json
{
  "insertInlineImage": {
    "location": { "index": 1 },
    "uri": "https://exemplo.com/diagrama.png",
    "objectSize": {
      "width":  { "magnitude": 400, "unit": "PT" },
      "height": { "magnitude": 250, "unit": "PT" }
    }
  }
}
```

---

## Cabeçalho e rodapé

`createHeader` / `createFooter` criam o segmento (se ainda não existir) e devolvem um `headerId`/`footerId` na resposta do `batchUpdate` — leia a resposta antes de escrever neles.

```json
{ "createHeader": { "type": "DEFAULT" } }
```

Com o ID em mãos (da resposta, campo `replies[].createHeader.headerId`), escreva usando `location.segmentId`:

```json
{
  "insertText": {
    "location": { "segmentId": "<headerId>", "index": 0 },
    "text": "Confidencial — uso interno"
  }
}
```
