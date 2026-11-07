---
name: google-docs
description: Lê, cria e edita documentos do Google Docs via chamadas curl nas APIs REST do Docs v1 e do Drive v3. Suporta ler o conteúdo de um documento em markdown, inspecionar sua estrutura (índices de caractere, estilos, marcadores), criar um documento novo — em branco ou copiando um existente como template — dentro de uma pasta do Drive, e preencher documentos com conteúdo. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN configurados com escopo de escrita (mesmas credenciais das skills google-drive, google-sheets e google-slides — veja ../google-drive/references/oauth-setup.md).
---

## Como usar

```
/google-docs ler <url ou ID do documento>
/google-docs estrutura <url ou ID do documento>
/google-docs criar "<título>" [base: <url ou ID>] [pasta: <url ou ID>]
/google-docs conteudo <url ou ID> — <descrição do que deve entrar no documento>
```

Exemplos:

```
/google-docs ler https://docs.google.com/document/d/1AbC.../edit
/google-docs estrutura 1AbC...
/google-docs criar "Contrato ACME" base: 1TplT... pasta: 1XyZ...
/google-docs conteudo 1NovD... — um título, dois parágrafos de introdução e uma lista de 3 itens
```

## Referências

- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](../google-drive/references/oauth-setup.md) — mesmo processo usado pela skill `google-drive`; veja em especial a seção sobre escopos de escrita.
- [Catálogo de requests do batchUpdate](references/batchupdate-requests.md) — leia antes de montar qualquer edição.

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0                                            |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md).

**Sobre o escopo:** diferente da skill `google-drive` em modo leitura, esta skill **escreve**. O escopo `drive.readonly` não é suficiente. O refresh token precisa ter sido gerado com:

- `https://www.googleapis.com/auth/documents` — criar e editar documentos
- `https://www.googleapis.com/auth/drive` — copiar o template, colocar o arquivo na pasta de destino e exportar o conteúdo em markdown

Se qualquer chamada retornar `403` com `insufficient authentication scopes` ou `Request had insufficient authentication scopes`, o refresh token ainda é o antigo (somente leitura): pare e oriente o usuário a regenerá-lo conforme a seção "Escopos de escrita" da referência de setup.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Portanto:

- **Ler respostas:** redirecione a saída do curl para um arquivo temporário e leia com a tool `Read`. Não tente extrair campos com `grep`/`cut` em JSON aninhado — só o `access_token` (formato fixo, uma linha) é seguro extrair assim.
- **Enviar corpos JSON:** escreva o JSON em um arquivo com a tool `Write` e envie com `curl -d @arquivo`. Nunca monte JSON grande inline no shell — o Git Bash no Windows quebra o escape de aspas.
- **Arquivos temporários:** use o diretório de scratchpad da sessão. Se não houver um informado, use `"${TEMP}/google-docs"` (crie com `mkdir -p`).

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS`:

1. Se começar com `ler` → **fluxo de leitura**.
2. Se começar com `estrutura` → **fluxo de inspeção de estrutura**.
3. Se começar com `criar` → **fluxo de criação**.
4. Se começar com `conteudo` (ou `conteúdo`) → **fluxo de conteúdo**.
5. Caso contrário, interprete o pedido em linguagem natural e mapeie para os fluxos acima, encadeando-os quando fizer sentido (ex: "crie um documento sobre X na pasta Y usando o template Z" = fluxo de criação seguido do fluxo de conteúdo). Se o pedido for ambíguo quanto ao template, à pasta de destino ou ao conteúdo, use `AskUserQuestion` antes de chamar a API.

**Regra que vale para todos os fluxos de edição:** nunca invente um índice de caractere. Todo `index`/`startIndex`/`endIndex` usado num `batchUpdate` vem do fluxo de inspeção de estrutura executado imediatamente antes — o documento muda de índices a cada request aplicada dentro do próprio lote (ver "Como os índices funcionam" em [references/batchupdate-requests.md](references/batchupdate-requests.md)).

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

- Documento: `https://docs.google.com/document/d/<ID>/edit` → `<ID>` (ignorar querystring e `#heading=...`)
- Pasta do Drive: `https://drive.google.com/drive/folders/<ID>` ou `https://drive.google.com/drive/u/0/folders/<ID>` → `<ID>`

Se o argumento não for uma URL, trate-o como o próprio ID. Se nenhum argumento foi informado, use `AskUserQuestion` para solicitar a URL ou ID do documento.

---

## Fluxo de leitura

Devolve o conteúdo já formatado, via export do Drive — não usa a API do Docs, então funciona mesmo com escopo só de leitura.

### Passo 1 — Extrair o ID e obter o access token

Ver passos comuns acima.

### Passo 2 — Buscar o título e exportar o conteúdo

```bash
DOC_ID="<id extraído>"
OUT="<dir temporário>/documento.md"
META="<dir temporário>/meta.json"

curl -s -G "https://www.googleapis.com/drive/v3/files/${DOC_ID}" \
  --data-urlencode "fields=name,webViewLink" \
  --data-urlencode "supportsAllDrives=true" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "$META"

curl -s -G "https://www.googleapis.com/drive/v3/files/${DOC_ID}/export" \
  --data-urlencode "mimeType=text/markdown" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "$OUT"
```

Se a exportação em `text/markdown` falhar (documento antigo ou tipo não suportado), repita com `mimeType=text/plain`.

### Passo 3 — Ler e exibir

Leia `$META` e `$OUT` com a tool `Read`. Se `$OUT` começar com `{` e contiver um campo `error`, exiba `error.message` (ex: documento não encontrado, sem permissão de acesso) e encerre.

Caso contrário, exiba o título (`name` de `$META`) seguido do conteúdo em markdown. Se o documento for muito longo, resuma ou mostre em partes em vez de despejar tudo de uma vez.

---

## Fluxo de inspeção de estrutura

Este fluxo é **pré-requisito obrigatório** do fluxo de conteúdo: é ele que revela os índices de caractere, os estilos aplicados e eventuais marcadores. Sem isso não há como editar com segurança.

### Passo 1 — Extrair o ID e obter o access token

Ver passos comuns acima.

### Passo 2 — Buscar a estrutura

```bash
DOC_ID="<id extraído>"
OUT="<dir temporário>/estrutura.json"

curl -s -G "https://docs.googleapis.com/v1/documents/${DOC_ID}" \
  --data-urlencode "fields=title,documentId,body(content(startIndex,endIndex,paragraph(paragraphStyle(namedStyleType),bullet,elements(startIndex,endIndex,textRun(content))),table(rows,columns,tableRows(tableCells(startIndex,endIndex))))),namedStyles(styles(namedStyleType))" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "$OUT"
```

O parâmetro `fields` é importante: sem ele a resposta traz toda a formatação de cada `textRun` e fica grande demais para ler.

### Passo 3 — Ler e interpretar

Leia `$OUT` com a tool `Read`. Se houver um campo `error`, exiba a mensagem (documento não encontrado, sem permissão, escopo insuficiente) e encerre.

Caso contrário, para cada item de `body.content`, extraia:

- **Faixa de índices** — `startIndex`/`endIndex` do elemento (parágrafo ou tabela).
- **Tipo** — `paragraph.paragraphStyle.namedStyleType` (`TITLE`, `HEADING_1`..`HEADING_6`, `NORMAL_TEXT`) ou, se for `table`, as dimensões (`rows` x `columns`) e o `startIndex` de cada célula em `tableRows[].tableCells[]`.
- **Texto** — concatenando os `textRun.content` dos `elements` do parágrafo.
- **Marcadores de substituição** — procure no texto por padrões tipo `{{algo}}`, `<<algo>>` ou `[[algo]]`. Se existirem, o documento foi feito para o padrão de substituição e o fluxo de conteúdo deve usar `replaceAllText`.

### Passo 4 — Exibir o resultado

```
**<title>** — <n> elementos | índice gravável: 1 a <endIndex do último elemento - 1>

Estrutura:
- [1–48] HEADING_1 — "Introdução"
- [49–210] NORMAL_TEXT — "O objetivo deste documento..."
- [211–350] TABLE 3x2

Marcadores encontrados: {{cliente}}, {{data}}  (ou "nenhum")
```

Ao final, se a inspeção foi feita como parte do fluxo de conteúdo, siga direto para ele sem esperar confirmação.

---

## Fluxo de criação

### Passo 1 — Interpretar os argumentos

Do argumento, extraia:

- **título** — texto entre aspas, ou o texto solto após `criar`
- **base** — valor após `base:` (URL ou ID do documento usado como template); opcional
- **pasta** — valor após `pasta:` (URL ou ID da pasta de destino); opcional

Se o título não foi informado, use `AskUserQuestion`. Se a pasta não foi informada, avise que o documento ficará na raiz do "Meu Drive" e siga (não bloqueie por isso).

### Passo 2 — Obter o access token

Ver passo comum acima.

### Passo 3a — Com template (recomendado): copiar

Copiar preserva estilos, cabeçalho/rodapé, fontes e eventuais marcadores do original, sem nenhuma conversão de formato. O `parents` já coloca a cópia na pasta certa — não é preciso mover depois.

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

Leia a resposta: `id` é o `documentId` do novo documento.

### Passo 3b — Sem template: criar em branco

```bash
BODY="<dir temporário>/create.json"   # {"title":"<título>"}

curl -s -X POST "https://docs.googleapis.com/v1/documents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @"$BODY" \
  -o "<dir temporário>/create-response.json"
```

O documento nasce na raiz do "Meu Drive", vazio.

Para movê-lo para a pasta de destino:

```bash
NEW_ID="<id do novo documento>"
FOLDER_ID="<id da pasta>"

curl -s -X PATCH \
  "https://www.googleapis.com/drive/v3/files/${NEW_ID}?addParents=${FOLDER_ID}&removeParents=root&supportsAllDrives=true&fields=id,parents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Length: 0" \
  -o "<dir temporário>/move-response.json"
```

### Passo 4 — Inspecionar e reportar

Rode o **fluxo de inspeção de estrutura** sobre o documento recém-criado (é o que habilita qualquer edição posterior) e informe o link:

```
Documento criado: https://docs.google.com/document/d/<NEW_ID>/edit
```

Se o pedido original também descrevia o conteúdo, siga direto para o fluxo de conteúdo.

---

## Fluxo de conteúdo

### Passo 1 — Inspecionar a estrutura

Execute o **fluxo de inspeção de estrutura** sobre o documento alvo. Obrigatório: os índices e os marcadores existentes vêm daí.

### Passo 2 — Escolher o padrão de montagem

| Situação identificada na inspeção | Padrão |
|---|---|
| Documento com marcadores (`{{...}}`), estilos e cabeçalho/rodapé já prontos | **Substituição** — `replaceAllText`, que não depende de índices e não é afetado por deslocamento |
| Documento em branco ou sem marcadores | **Inserção posicional** — `insertText` + `updateParagraphStyle` / `createParagraphBullets`, usando os índices lidos na estrutura |

Na dúvida, prefira **substituição**: ela preserva qualquer coisa que o template tenha além dos marcadores e não sofre com deslocamento de índices.

Leia [references/batchupdate-requests.md](references/batchupdate-requests.md) antes de montar as requests — em especial a seção "Como os índices funcionam" se for usar inserção posicional.

### Passo 3 — Montar o corpo do batchUpdate

Escreva o JSON em arquivo com a tool `Write`:

```json
{ "requests": [ ... ] }
```

Regras que evitam a maioria dos erros:

- **A ordem importa e os índices se deslocam.** Cada `insertText` aplicado empurra para frente todos os índices seguintes do mesmo lote. Ao inserir em várias posições no mesmo lote, ordene as requests da **maior posição para a menor**, ou use `endOfSegmentLocation` para anexar ao fim (imune a deslocamento).
- **É atômico.** Se uma request falhar, nenhuma é aplicada. Não há estado parcial para limpar, mas também não adianta reenviar o lote inteiro sem corrigir o erro.
- **`endIndex` de um parágrafo inclui o `\n` final** — para apagar um parágrafo sem fundi-lo com o seguinte, use `endIndex - 1` em `deleteContentRange`.
- **Depois de inserir uma tabela**, os índices do restante do documento mudam de forma difícil de prever — releia a estrutura antes de montar o próximo lote.

### Passo 4 — Enviar

```bash
DOC_ID="<id>"
BODY="<dir temporário>/batch.json"

curl -s -X POST \
  "https://docs.googleapis.com/v1/documents/${DOC_ID}:batchUpdate" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @"$BODY" \
  -o "<dir temporário>/batch-response.json"
```

### Passo 5 — Verificar e reportar

Leia a resposta. Se houver `error`, exiba `error.message` e diga qual request do lote provavelmente causou o problema (a mensagem costuma citar o índice ou o número da request). Erros mais comuns:

| Mensagem | Causa |
|---|---|
| `Index ... must be less than the end index of the referenced segment` | índice além do fim do corpo — a estrutura foi lida antes de uma inserção anterior do mesmo lote, ou o documento mudou entre a leitura e o envio |
| `The starting index must be less than the ending index` | intervalo invertido ou vazio em `deleteContentRange` |
| `Invalid requests[N]: ...` | request malformada; a mensagem cita o índice do lote |
| `insufficient authentication scopes` | refresh token sem escopo de escrita (ver seção de escopo acima) |

Em caso de sucesso, informe o que foi feito e o link:

```
Documento atualizado: https://docs.google.com/document/d/<ID>/edit
```
