---
name: google-slides
description: Cria e edita apresentações do Google Slides via chamadas curl nas APIs REST do Slides v1 e do Drive v3. Suporta inspecionar a estrutura de uma apresentação (layouts, placeholders e marcadores), criar uma apresentação nova — em branco ou copiando uma existente como template — dentro de uma pasta do Drive, e preencher slides com conteúdo. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN configurados com escopo de escrita (mesmas credenciais das skills google-drive e google-sheets — veja ../google-drive/references/oauth-setup.md).
---

## Como usar

```
/google-slides estrutura <url ou ID da apresentação>
/google-slides criar "<título>" [base: <url ou ID>] [pasta: <url ou ID>]
/google-slides conteudo <url ou ID> — <descrição do que deve entrar nos slides>
```

Exemplos:

```
/google-slides estrutura https://docs.google.com/presentation/d/1AbC.../edit
/google-slides criar "Workshop DevOps - Módulo 3" base: 1AbC... pasta: 1XyZ...
/google-slides conteudo 1NovA... — 5 slides sobre pipelines de CI/CD, um tópico por slide
```

## Referências

- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](../google-drive/references/oauth-setup.md) — mesmo processo usado pela skill `google-drive`; veja em especial a seção sobre escopos de escrita.
- [Catálogo de requests do batchUpdate](references/batchupdate-requests.md) — leia antes de montar qualquer conteúdo.

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0                                            |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md).

**Sobre o escopo:** diferente das skills `google-drive` e `google-sheets`, esta skill **escreve**. O escopo `drive.readonly` não é suficiente. O refresh token precisa ter sido gerado com:

- `https://www.googleapis.com/auth/presentations` — criar e editar apresentações
- `https://www.googleapis.com/auth/drive` — copiar o template e colocar o arquivo na pasta de destino

Se qualquer chamada retornar `403` com `insufficient authentication scopes` ou `Request had insufficient authentication scopes`, o refresh token ainda é o antigo (somente leitura): pare e oriente o usuário a regenerá-lo conforme a seção "Escopos de escrita" da referência de setup.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Portanto:

- **Ler respostas:** redirecione a saída do curl para um arquivo temporário e leia com a tool `Read`. Não tente extrair campos com `grep`/`cut` em JSON aninhado — só o `access_token` (formato fixo, uma linha) é seguro extrair assim.
- **Enviar corpos JSON:** escreva o JSON em um arquivo com a tool `Write` e envie com `curl -d @arquivo`. Nunca monte JSON grande inline no shell — o Git Bash no Windows quebra o escape de aspas.
- **Arquivos temporários:** use o diretório de scratchpad da sessão. Se não houver um informado, use `"${TEMP}/google-slides"` (crie com `mkdir -p`).

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS`:

1. Se começar com `estrutura` → **fluxo de inspeção de estrutura**.
2. Se começar com `criar` → **fluxo de criação**.
3. Se começar com `conteudo` (ou `conteúdo`) → **fluxo de conteúdo**.
4. Caso contrário, interprete o pedido em linguagem natural e mapeie para os fluxos acima, encadeando-os quando fizer sentido (ex: "crie uma apresentação sobre X na pasta Y usando o template Z" = fluxo de criação seguido do fluxo de conteúdo). Se o pedido for ambíguo quanto ao template, à pasta de destino ou ao conteúdo, use `AskUserQuestion` antes de chamar a API.

**Regra que vale para todos os fluxos:** nunca invente `objectId` de layout, de placeholder ou de slide. Todo ID usado em um `batchUpdate` ou vem do fluxo de inspeção de estrutura, ou foi definido por você na mesma requisição (via `objectId` do `createSlide` / mapa `objectIds` do `duplicateObject`).

---

## Passo comum — Obter um access token

Antes de qualquer chamada às APIs, troque o refresh token por um access token (válido por ~1 hora):

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

O access token vale para todas as chamadas do mesmo fluxo — obtenha uma vez só e reaproveite. Como o shell não mantém estado entre chamadas da tool `Bash`, ou faça todas as chamadas de um fluxo no mesmo bloco, ou repita o bloco de token no início de cada bloco.

---

## Passo comum — Extrair IDs de URLs

- Apresentação: `https://docs.google.com/presentation/d/<ID>/edit` → `<ID>` (ignorar querystring e `#slide=...`)
- Pasta do Drive: `https://drive.google.com/drive/folders/<ID>` ou `https://drive.google.com/drive/u/0/folders/<ID>` → `<ID>`

Se o argumento não for uma URL, trate-o como o próprio ID.

---

## Fluxo de inspeção de estrutura

Este fluxo é **pré-requisito obrigatório** do fluxo de conteúdo: é ele que revela os `objectId` dos layouts e os placeholders disponíveis. Sem isso não há como criar slides no estilo do template.

### Passo 1 — Extrair o ID e obter o access token

Ver passos comuns acima.

### Passo 2 — Buscar a estrutura

```bash
PRESENTATION_ID="<id extraído>"
OUT="<dir temporário>/estrutura.json"

curl -s -G "https://slides.googleapis.com/v1/presentations/${PRESENTATION_ID}" \
  --data-urlencode "fields=title,presentationId,pageSize,layouts(objectId,layoutProperties(name,displayName),pageElements(objectId,shape(placeholder(type,index)))),slides(objectId,slideProperties(layoutObjectId),pageElements(objectId,shape(placeholder(type,index),text(textElements(textRun(content))))))" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "$OUT"
```

O parâmetro `fields` é importante: sem ele a resposta traz toda a formatação de cada elemento e fica grande demais para ler.

### Passo 3 — Ler e interpretar

Leia `$OUT` com a tool `Read`. Se houver um campo `error`, exiba a mensagem (apresentação não encontrada, sem permissão, escopo insuficiente) e encerre.

Caso contrário, extraia:

- **Layouts disponíveis** — para cada item de `layouts`: `objectId`, `layoutProperties.displayName` (nome legível, ex: "Título e corpo") e a lista de placeholders (`pageElements[].shape.placeholder`, com `type` e `index`). São esses `type`/`index` que vão no `placeholderIdMappings`.
- **Slides existentes** — para cada item de `slides`: `objectId`, o layout que ele usa (`slideProperties.layoutObjectId`) e o texto de cada elemento (concatenando os `textRun.content`).
- **Marcadores de substituição** — procure no texto dos slides por padrões tipo `{{algo}}`, `<<algo>>` ou `[[algo]]`. Se existirem, o template foi feito para o padrão de duplicação e o fluxo de conteúdo deve usar `duplicateObject` + `replaceAllText`.
- **Proporção** — `pageSize` em EMU (16:9 padrão = 9144000 x 5143500).

### Passo 4 — Exibir o resultado

```
**<title>** — <n> slides | <largura>x<altura> EMU

Layouts:
- <displayName> (`<objectId>`) — placeholders: TITLE[0], BODY[0]

Slides:
1. `<objectId>` — layout <displayName> — "<primeiras palavras do título>"

Marcadores encontrados: {{titulo}}, {{bullet1}}  (ou "nenhum")
```

Ao final, se a inspeção foi feita como parte do fluxo de conteúdo, siga direto para ele sem esperar confirmação.

---

## Fluxo de criação

### Passo 1 — Interpretar os argumentos

Do argumento, extraia:

- **título** — texto entre aspas, ou o texto solto após `criar`
- **base** — valor após `base:` (URL ou ID da apresentação usada como template); opcional
- **pasta** — valor após `pasta:` (URL ou ID da pasta de destino); opcional

Se o título não foi informado, use `AskUserQuestion`. Se a pasta não foi informada, avise que a apresentação ficará na raiz do "Meu Drive" e siga (não bloqueie por isso).

### Passo 2 — Obter o access token

Ver passo comum acima.

### Passo 3a — Com template (recomendado): copiar

Copiar preserva tema, masters, layouts, fontes e cores do original, sem nenhuma conversão de formato. O `parents` já coloca a cópia na pasta certa — não é preciso mover depois.

```bash
TEMPLATE_ID="<id da base>"
BODY="<dir temporário>/copy.json"   # escrito com a tool Write
# conteúdo de copy.json: {"name":"<título>","parents":["<folderId>"]}
# sem pasta de destino, omita a chave "parents"

curl -s -X POST \
  "https://www.googleapis.com/drive/v3/files/${TEMPLATE_ID}/copy?supportsAllDrives=true&fields=id,name,webViewLink" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @"$BODY" \
  -o "<dir temporário>/copy-response.json"
```

Leia a resposta: `id` é o `presentationId` da nova apresentação.

### Passo 3b — Sem template: criar em branco

```bash
BODY="<dir temporário>/create.json"   # {"title":"<título>"}

curl -s -X POST "https://slides.googleapis.com/v1/presentations" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @"$BODY" \
  -o "<dir temporário>/create-response.json"
```

A apresentação nasce na raiz do "Meu Drive" e já vem com um slide inicial — confira `slides[]` na resposta e decida no fluxo de conteúdo se reaproveita esse slide ou o remove com `deleteObject`.

Para movê-la para a pasta de destino:

```bash
NEW_ID="<id da nova apresentação>"
FOLDER_ID="<id da pasta>"

curl -s -X PATCH \
  "https://www.googleapis.com/drive/v3/files/${NEW_ID}?addParents=${FOLDER_ID}&removeParents=root&supportsAllDrives=true&fields=id,parents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Length: 0" \
  -o "<dir temporário>/move-response.json"
```

### Passo 4 — Inspecionar e reportar

Rode o **fluxo de inspeção de estrutura** sobre a apresentação recém-criada (é o que habilita qualquer edição posterior) e informe o link:

```
Apresentação criada: https://docs.google.com/presentation/d/<NEW_ID>/edit
```

Se o pedido original também descrevia o conteúdo, siga direto para o fluxo de conteúdo.

---

## Fluxo de conteúdo

### Passo 1 — Inspecionar a estrutura

Execute o **fluxo de inspeção de estrutura** sobre a apresentação alvo. Obrigatório: os `objectId` de layouts e slides-modelo vêm daí.

### Passo 2 — Escolher o padrão de montagem

| Situação identificada na inspeção | Padrão |
|---|---|
| Slides-modelo com marcadores (`{{...}}`), arte, ícones, caixas posicionadas | **Duplicação** — `duplicateObject` do slide-modelo + `replaceAllText` restrito à cópia via `pageObjectIds` |
| Só layouts limpos (título, título+corpo, seção), sem slides-modelo | **Layout** — `createSlide` com `slideLayoutReference.layoutId` + `placeholderIdMappings` + `insertText` |

Na dúvida, prefira **duplicação**: ela preserva qualquer coisa que o template tenha além dos placeholders.

Leia [references/batchupdate-requests.md](references/batchupdate-requests.md) antes de montar as requests.

### Passo 3 — Montar o corpo do batchUpdate

Escreva o JSON em arquivo com a tool `Write`:

```json
{ "requests": [ ... ] }
```

Regras que evitam a maioria dos erros:

- **A ordem importa.** As requests são aplicadas em sequência. Crie/duplique primeiro, insira texto depois, e **delete os slides-modelo por último** — se deletar antes, o `duplicateObject` não encontra a origem.
- **É atômico.** Se uma request falhar, nenhuma é aplicada. Não há estado parcial para limpar, mas também não adianta reenviar o lote inteiro sem corrigir o erro.
- **`objectId` definido por você** deve ter de 5 a 50 caracteres, usar só `[a-zA-Z0-9_-]` e ser único na apresentação. IDs curtos como `s1` são rejeitados.
- **Não misture** um `insertText` em um placeholder criado por `createSlide` sem ter declarado o `placeholderIdMappings` correspondente — sem o mapeamento você não tem o `objectId` para escrever nele.

### Passo 4 — Enviar

```bash
PRESENTATION_ID="<id>"
BODY="<dir temporário>/batch.json"

curl -s -X POST \
  "https://slides.googleapis.com/v1/presentations/${PRESENTATION_ID}:batchUpdate" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @"$BODY" \
  -o "<dir temporário>/batch-response.json"
```

### Passo 5 — Verificar e reportar

Leia a resposta. Se houver `error`, exiba `error.message` e diga qual request do lote provavelmente causou o problema (a mensagem costuma citar o `objectId` ou o índice). Erros mais comuns:

| Mensagem | Causa |
|---|---|
| `Invalid requests[N]...: The object ID ... is not valid` | `objectId` fora das regras (curto demais ou com caractere inválido) |
| `... does not exist` | ID de layout/slide inventado, ou objeto deletado antes de ser usado |
| `Invalid requests[N].insertText: The object ... is not a text-based object` | tentativa de escrever em algo que não é caixa de texto/placeholder |
| `insufficient authentication scopes` | refresh token sem escopo de escrita (ver seção de escopo acima) |

Em caso de sucesso, informe o que foi feito e o link:

```
<n> slides criados em "<título>": https://docs.google.com/presentation/d/<ID>/edit
```
