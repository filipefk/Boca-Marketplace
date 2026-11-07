---
name: google-drive
description: Acessa o Google Drive via chamadas curl na API REST v3. Por enquanto suporta listar o conteúdo de uma pasta a partir de uma URL ou ID. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN configurados (veja references/oauth-setup.md).
---

## Como usar

```
/google-drive listar <url ou ID da pasta>
```

## Referências

- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](references/oauth-setup.md)

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0 (tipo "Aplicativo para computador")        |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [references/oauth-setup.md](references/oauth-setup.md).

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS`:

1. Se começar com `listar` → **fluxo de listagem de pasta**.
2. Caso contrário → informe que, por enquanto, a skill só suporta o comando `listar` e mostre o exemplo de uso.

---

## Passo comum — Obter um access token

Antes de qualquer chamada à API do Drive, troque o refresh token por um access token (válido por ~1 hora):

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

Se `ACCESS_TOKEN` vier vazio, exiba o conteúdo de `$TOKEN_RESPONSE` como erro e encerre (provavelmente o refresh token expirou ou foi revogado — oriente o usuário a refazer o passo de autorização em [references/oauth-setup.md](references/oauth-setup.md)).

---

## Fluxo de listagem de pasta

### Passo 1 — Extrair o ID da pasta

Analise o argumento (após remover o prefixo `listar`). Se for uma URL do Drive, extraia o ID:

- `https://drive.google.com/drive/folders/<ID>` → `<ID>`
- `https://drive.google.com/drive/folders/<ID>?usp=sharing` → `<ID>` (ignorar querystring)
- `https://drive.google.com/drive/u/0/folders/<ID>` → `<ID>`
- Se não for uma URL, trate o argumento inteiro como o próprio ID da pasta.

Se nenhum argumento foi informado, use `AskUserQuestion` para solicitar a URL ou ID da pasta.

### Passo 2 — Obter o access token

Execute o passo comum acima.

### Passo 3 — Listar os arquivos da pasta

```bash
FOLDER_ID="<id extraído no passo 1>"

curl -s -G "https://www.googleapis.com/drive/v3/files" \
  --data-urlencode "q='${FOLDER_ID}' in parents and trashed=false" \
  --data-urlencode "fields=files(id,name,mimeType,modifiedTime,webViewLink)" \
  --data-urlencode "pageSize=100" \
  --data-urlencode "supportsAllDrives=true" \
  --data-urlencode "includeItemsFromAllDrives=true" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Passo 4 — Exibir o resultado

Se a resposta contiver um campo `error`, exiba a mensagem de erro (ex: pasta não encontrada, sem permissão de acesso) e encerre.

Caso contrário, para cada item em `files`, exibir:

```
- [<name>](<webViewLink>) — <"Pasta" se mimeType for "application/vnd.google-apps.folder", senão "Arquivo"> | Modificado em: <modifiedTime>
```

Se a lista estiver vazia, informar que a pasta não possui arquivos (ou que a conta autenticada não tem acesso a ela).
