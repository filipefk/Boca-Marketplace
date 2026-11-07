# Catálogo de requests do `presentations.batchUpdate`

Endpoint: `POST https://slides.googleapis.com/v1/presentations/{presentationId}:batchUpdate`
Corpo: `{"requests": [ <request>, <request>, ... ]}`

Todas as requests do lote são aplicadas em ordem e de forma atômica: se uma falhar, nenhuma é aplicada.

## Regras de `objectId`

IDs que você define (em `createSlide`, no mapa `objectIds` do `duplicateObject`, em `createShape`) precisam:

- ter de **5 a 50 caracteres**;
- usar apenas `[a-zA-Z0-9_-]`;
- ser únicos na apresentação inteira (não só no slide).

`s1` falha. `slide_01` e `s01_titulo` funcionam. Use nomes previsíveis (`slide_03`, `slide_03_titulo`, `slide_03_corpo`) — isso permite escrever o `insertText` no mesmo lote sem precisar ler a resposta.

## Unidades

Medidas usam EMU (English Metric Units):

| Referência | EMU |
|---|---|
| 1 polegada | 914400 |
| 1 ponto (pt) | 12700 |
| 1 cm | 360000 |
| Slide 16:9 padrão | 9144000 x 5143500 |
| Slide 4:3 padrão | 9144000 x 6858000 |

Confirme o tamanho real em `pageSize`, retornado pelo `presentations.get`.

---

## Padrão A — Duplicação (template com slides-modelo)

Use quando o template tem slides prontos com arte, ícones ou caixas posicionadas, geralmente com marcadores de texto tipo `{{titulo}}`.

### `duplicateObject`

Duplica um slide inteiro (com todo o conteúdo) logo **após** o original. O mapa `objectIds` renomeia os objetos da cópia — a chave é o ID do objeto original, o valor é o novo ID. Inclua o próprio slide no mapa: sem isso a cópia recebe um ID aleatório e você não consegue restringir o `replaceAllText` a ela no mesmo lote.

```json
{
  "duplicateObject": {
    "objectId": "g1a2b3c4d5_0_0",
    "objectIds": { "g1a2b3c4d5_0_0": "slide_topico_01" }
  }
}
```

Os demais elementos da cópia (não listados no mapa) recebem IDs gerados automaticamente — o que não é problema no padrão de duplicação, porque o texto é trocado por conteúdo, não por ID.

### `replaceAllText`

Troca texto em toda a apresentação, ou apenas nas páginas listadas em `pageObjectIds`. **Sempre informe `pageObjectIds`** ao preencher uma cópia — sem isso a substituição também atinge o slide-modelo e os demais slides já preenchidos.

```json
{
  "replaceAllText": {
    "containsText": { "text": "{{titulo}}", "matchCase": true },
    "replaceText": "Pipelines de CI/CD",
    "pageObjectIds": ["slide_topico_01"]
  }
}
```

### `replaceAllShapesWithImage`

Troca as formas que contêm um texto por uma imagem (útil para placeholders de imagem no template). A URL precisa ser publicamente acessível.

```json
{
  "replaceAllShapesWithImage": {
    "containsText": { "text": "{{imagem}}", "matchCase": true },
    "imageUrl": "https://exemplo.com/diagrama.png",
    "imageReplaceMethod": "CENTER_INSIDE",
    "pageObjectIds": ["slide_topico_01"]
  }
}
```

### Ordem típica do lote

1. `duplicateObject` do slide-modelo, uma vez por slide desejado (cada um com seu `objectIds`);
2. `replaceAllText` por slide criado, restrito via `pageObjectIds`;
3. `updateSlidesPosition` se a ordem final precisar mudar;
4. `deleteObject` dos slides-modelo, **por último**.

---

## Padrão B — Layout (template com layouts limpos)

Use quando não há slides-modelo, só os layouts do tema.

### `createSlide`

`slideLayoutReference` aceita `layoutId` (o `objectId` de um layout real do template — preferível) ou `predefinedLayout` (nome genérico, que só funciona se o master tiver aquele layout).

`placeholderIdMappings` é o que dá nome aos placeholders da cópia, permitindo escrever neles em seguida. O `type` e o `index` vêm do `presentations.get` (`layouts[].pageElements[].shape.placeholder`).

```json
{
  "createSlide": {
    "objectId": "slide_03",
    "insertionIndex": 2,
    "slideLayoutReference": { "layoutId": "p3" },
    "placeholderIdMappings": [
      { "layoutPlaceholder": { "type": "TITLE", "index": 0 }, "objectId": "slide_03_titulo" },
      { "layoutPlaceholder": { "type": "BODY",  "index": 0 }, "objectId": "slide_03_corpo" }
    ]
  }
}
```

Tipos de placeholder mais comuns: `TITLE`, `CENTERED_TITLE`, `SUBTITLE`, `BODY`, `SLIDE_NUMBER`. `insertionIndex` é opcional (sem ele, o slide vai para o fim).

Valores de `predefinedLayout`: `BLANK`, `TITLE`, `TITLE_AND_BODY`, `TITLE_ONLY`, `SECTION_HEADER`, `SECTION_TITLE_AND_DESCRIPTION`, `ONE_COLUMN_TEXT`, `MAIN_POINT`, `BIG_NUMBER`, `CAPTION_ONLY`, `TITLE_AND_TWO_COLUMNS`.

### `insertText`

```json
{
  "insertText": {
    "objectId": "slide_03_corpo",
    "text": "Build automatizado\nTestes em cada push\nDeploy contínuo",
    "insertionIndex": 0
  }
}
```

Em um placeholder de corpo, cada `\n` vira um item da lista. Para transformar em bullets explícitos, use `createParagraphBullets`:

```json
{
  "createParagraphBullets": {
    "objectId": "slide_03_corpo",
    "textRange": { "type": "ALL" },
    "bulletPreset": "BULLET_DISC_CIRCLE_SQUARE"
  }
}
```

---

## Requests auxiliares

### `deleteObject`

Remove um slide ou um elemento. Sempre por último no lote quando o alvo for origem de duplicação.

```json
{ "deleteObject": { "objectId": "g1a2b3c4d5_0_0" } }
```

### `updateSlidesPosition`

Move slides para uma posição. `insertionIndex` é a posição final (base 0) do primeiro slide da lista.

```json
{ "updateSlidesPosition": { "slideObjectIds": ["slide_topico_01"], "insertionIndex": 1 } }
```

### `createShape` (caixa de texto livre)

Quando não há placeholder adequado. Requer posição e tamanho explícitos.

```json
{
  "createShape": {
    "objectId": "slide_03_nota",
    "shapeType": "TEXT_BOX",
    "elementProperties": {
      "pageObjectId": "slide_03",
      "size": {
        "width":  { "magnitude": 4000000, "unit": "EMU" },
        "height": { "magnitude": 800000,  "unit": "EMU" }
      },
      "transform": {
        "scaleX": 1, "scaleY": 1,
        "translateX": 500000, "translateY": 3800000,
        "unit": "EMU"
      }
    }
  }
}
```

Depois use `insertText` no mesmo `objectId`.

### `createImage`

A URL precisa ser publicamente acessível (o Google baixa a imagem no momento da chamada), em PNG/JPEG/GIF e até 50 MB. Links do Drive que exigem login **não funcionam**.

```json
{
  "createImage": {
    "objectId": "slide_03_img",
    "url": "https://exemplo.com/diagrama.png",
    "elementProperties": {
      "pageObjectId": "slide_03",
      "size": {
        "width":  { "magnitude": 3000000, "unit": "EMU" },
        "height": { "magnitude": 2000000, "unit": "EMU" }
      },
      "transform": {
        "scaleX": 1, "scaleY": 1,
        "translateX": 5000000, "translateY": 1500000,
        "unit": "EMU"
      }
    }
  }
}
```

### Anotações do apresentador (speaker notes)

O texto vai em uma caixa própria da página de anotações. O ID dela vem do `presentations.get`, no campo `slides[].slideProperties.notesPage.notesProperties.speakerNotesObjectId` — esse campo **não** está incluído na máscara `fields` usada no fluxo de inspeção padrão, então faça uma chamada específica quando precisar:

```
fields=slides(objectId,slideProperties/notesPage/notesProperties/speakerNotesObjectId)
```

Com o ID em mãos, é um `insertText` normal:

```json
{ "insertText": { "objectId": "<speakerNotesObjectId>", "text": "Falar sobre o custo de rollback manual.", "insertionIndex": 0 } }
```

### Formatação de texto

`updateTextStyle` altera fonte, tamanho, cor e negrito de um intervalo. `fields` indica quais propriedades aplicar (é obrigatório).

```json
{
  "updateTextStyle": {
    "objectId": "slide_03_titulo",
    "textRange": { "type": "ALL" },
    "style": {
      "bold": true,
      "fontSize": { "magnitude": 28, "unit": "PT" },
      "foregroundColor": { "opaqueColor": { "rgbColor": { "red": 0.1, "green": 0.1, "blue": 0.1 } } }
    },
    "fields": "bold,fontSize,foregroundColor"
  }
}
```

Evite formatar manualmente quando o template já define o estilo no layout — o ganho do template é justamente não precisar disso.
