# Configuração da credencial OAuth 2.0 (setup único, feito pelo usuário)

Esta skill não executa este fluxo sozinha — ele é feito manualmente uma única vez pelo usuário no navegador, fora do Claude Code. Depois de concluído, as três variáveis de ambiente ficam configuradas e a skill funciona sem intervenção.

## Por que uma credencial nova (e não a do n8n)

O Client ID/Secret usado no n8n é válido, mas o OAuth client cadastrado no Google Cloud só aceita os redirect URIs que estiverem na lista dele (provavelmente só o callback do n8n). Para gerar um refresh token novo pra esta skill é preciso um redirect URI diferente. Criar uma credencial dedicada evita qualquer alteração na credencial de produção do n8n.

## Passo 1 — Criar o OAuth Client no Google Cloud Console

1. Acesse https://console.cloud.google.com/apis/credentials no mesmo projeto usado pelo n8n (ou em outro, se preferir isolar).
2. Se ainda não houver, configure a "Tela de consentimento OAuth" (OAuth consent screen) com o escopo `https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/presentations https://www.googleapis.com/auth/documents https://www.googleapis.com/auth/spreadsheets`.
3. Em "Criar credenciais" → "ID do cliente OAuth", escolha o tipo **"Aplicativo da Web"** (Web application).
   > Não use "Aplicativo para computador" (Desktop app): esse tipo só aceita redirect URIs de loopback automáticos e não permite cadastrar o redirect do OAuth Playground, causando o erro `redirect_uri_mismatch`.
4. Em **"URIs de redirecionamento autorizados"**, adicione: `https://developers.google.com/oauthplayground`
5. Anote o **Client ID** e o **Client Secret** gerados.

## Passo 2 — Habilitar as APIs

Em https://console.cloud.google.com/apis/library, ative as APIs que serão usadas (se ainda não estiverem ativas para o projeto):

- **Google Drive API** — skill `google-drive` (e usada também pelas outras)
- **Google Sheets API** — skill `google-sheets`
- **Google Slides API** — skill `google-slides`
- **Google Docs API** — skill `google-docs`

## Passo 3 — Gerar o refresh token via OAuth Playground

1. Acesse https://developers.google.com/oauthplayground
2. Clique no ícone de engrenagem (canto superior direito) e marque **"Use your own OAuth credentials"**. Informe o Client ID e Client Secret do Passo 1.
3. Na lista de escopos à esquerda, informe manualmente:
```
https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/presentations https://www.googleapis.com/auth/documents https://www.googleapis.com/auth/spreadsheets
```
4. Clique em **"Authorize APIs"** e faça login com a conta Google dona dos arquivos/pastas do Drive que serão acessados.
5. Na tela seguinte ("Step 2"), clique em **"Exchange authorization code for tokens"**.
6. Copie o valor de **Refresh token** exibido.

## Escopos de escrita (necessário para as skills `google-slides` e `google-docs`)

As skills `google-drive` e `google-sheets` só leem, e `drive.readonly` basta para as duas. As skills `google-slides` e `google-docs` **criam e editam arquivos**, e para isso o refresh token precisa ser regenerado com escopos maiores.

Refaça o Passo 3 informando, no campo de escopos do OAuth Playground, esta lista (separada por espaços):

```
https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/presentations https://www.googleapis.com/auth/documents https://www.googleapis.com/auth/spreadsheets
```

| Escopo | Para quê |
|---|---|
| `drive` | copiar o template (apresentação ou documento) e colocar o arquivo na pasta de destino |
| `presentations` | criar a apresentação e aplicar `batchUpdate` nos slides |
| `documents` | criar o documento e aplicar `batchUpdate` no conteúdo |
| `spreadsheets` | opcional — só se a skill `google-sheets` passar a escrever também |

O novo refresh token **substitui** o antigo na variável `GOOGLE_REFRESH_TOKEN`: como `drive` engloba `drive.readonly`, um único token atende as quatro skills. Guarde o token antigo até confirmar que as skills de leitura continuam funcionando.

### Sobre o prazo de validade

`https://www.googleapis.com/auth/drive` é um escopo **restrito** pelo Google. Duas consequências práticas:

- Com o app em **"Testing"** na tela de consentimento, o refresh token expira em **7 dias** — você teria que regenerá-lo toda semana.
- Publicando o app em **"Em produção"**, o refresh token deixa de expirar. Como o app não é verificado, o login vai exibir a tela "O Google não verificou este app" — clique em "Avançado" → "Acessar <nome do app> (não seguro)". Para uso pessoal/interno isso é suficiente; a verificação formal só é exigida para distribuir o app a terceiros.

Alternativa mais restrita: trocar `drive` por `drive.readonly` + `drive.file` (escopo por arquivo criado pelo app). Reduz o alcance do token, mas copiar um template para dentro de uma pasta preexistente pode falhar por falta de permissão sobre a pasta — teste antes de adotar.

## Passo 4 — Configurar as variáveis de ambiente

Defina as três variáveis no ambiente onde o Claude Code roda (ex: `.env`, variáveis de sistema do Windows, ou perfil do PowerShell):

```
GOOGLE_CLIENT_ID=<client id do passo 1>
GOOGLE_CLIENT_SECRET=<client secret do passo 1>
GOOGLE_REFRESH_TOKEN=<refresh token do passo 3>
```

## Observações

- O refresh token não expira por tempo, mas é revogado se o usuário remover o acesso do app em https://myaccount.google.com/permissions, se ficar 6 meses sem uso, ou se o OAuth consent screen estiver em modo "Testing" e o usuário for removido da lista de testadores.
- Caso o app esteja em modo "Testing" (comum para uso pessoal/interno), o refresh token expira em 7 dias. Para uso contínuo, publique o app na tela de consentimento (não precisa de verificação do Google se o escopo for só `drive.readonly` e o uso for restrito).
- Para acessar outras pastas/arquivos, a conta autorizada no Passo 3 precisa ter permissão de leitura sobre eles no Drive.
