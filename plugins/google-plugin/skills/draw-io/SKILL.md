---
name: draw-io
description: Lê, edita e cria diagramas .drawio (diagrams.net) armazenados no Google Drive, manipulando diretamente o XML do mxGraph. Requer GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET e GOOGLE_REFRESH_TOKEN com escopo de escrita (mesmas credenciais das skills google-drive, google-slides e google-docs — veja ../google-drive/references/oauth-setup.md).
---

## Como usar

```
/draw-io ler <url ou ID do arquivo>
/draw-io editar <url ou ID do arquivo> — <descrição da alteração>
/draw-io criar "<nome>" [pasta: <url ou ID>] [— <descrição do conteúdo inicial>]
```

Exemplos:

```
/draw-io ler https://drive.google.com/file/d/1AbC.../view
/draw-io editar 1AbC... — adicionar uma caixa "Novo passo" conectada à caixa "Início"
/draw-io criar "Fluxo de onboarding" pasta: 1XyZ... — 3 etapas em sequência: Cadastro, Verificação, Ativação
```

## Referências

- [Configuração da credencial OAuth 2.0 e obtenção do refresh token](../google-drive/references/oauth-setup.md) — mesmo processo usado pelas skills `google-drive`, `google-slides` e `google-docs`; veja em especial a seção "Escopos de escrita". O escopo `drive` já cobre esta skill — não é preciso nenhum escopo adicional.
- [Formato XML do draw.io (mxGraph)](references/drawio-xml-format.md) — leia antes de montar ou interpretar qualquer `mxCell`. Cobre estrutura, compressão do conteúdo e a decisão de sempre gravar sem compressão.

## Variáveis de ambiente

| Variável               | Descrição                                                        |
|-------------------------|-------------------------------------------------------------------|
| `GOOGLE_CLIENT_ID`      | Client ID do OAuth 2.0                                            |
| `GOOGLE_CLIENT_SECRET`  | Client Secret correspondente                                      |
| `GOOGLE_REFRESH_TOKEN`  | Refresh token obtido uma única vez (ver referência de setup)       |

Se alguma dessas variáveis não estiver definida, informe ao usuário e aponte para [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md).

**Sobre o escopo:** esta skill **lê e escreve** arquivos no Drive. O escopo `drive.readonly` não é suficiente — o refresh token precisa ter sido gerado com `https://www.googleapis.com/auth/drive` (ver "Escopos de escrita" na referência de setup). Se qualquer chamada retornar `403` com `insufficient authentication scopes`, oriente o usuário a regenerar o refresh token.

**Sobre Drives Compartilhados (Shared Drives):** por padrão, a API do Drive **ignora** arquivos e pastas que estão dentro de um Drive Compartilhado — mesmo que o usuário autenticado tenha acesso normal a eles pelo navegador. Sem os parâmetros corretos, `files.get` retorna `404 File not found` e `files.list` retorna uma lista vazia, mesmo quando o item existe e é acessível. Por isso, **toda** chamada desta skill à API do Drive (`files.get`, `files.list`, upload/PATCH, create) deve incluir `supportsAllDrives=true`; chamadas de `files.list` devem incluir também `includeItemsFromAllDrives=true` e `corpora=allDrives`. Os snippets abaixo já vêm com esses parâmetros — não os remova.

## Ferramentas disponíveis no ambiente

Não há `jq` nem `python` no PATH desta máquina. Portanto:

- **Ler respostas da API:** redirecione a saída do curl para um arquivo temporário e leia com a tool `Read`. Não tente extrair campos com `grep`/`cut` em JSON aninhado — só o `access_token` (formato fixo, uma linha) é seguro extrair assim.
- **Enviar corpos JSON:** escreva o JSON em um arquivo com a tool `Write` e envie com `curl -d @arquivo`. Nunca monte JSON grande inline no shell — o Git Bash no Windows quebra o escape de aspas.
- **(Des)comprimir o XML do diagrama:** o conteúdo de cada `<diagram>` normalmente vem comprimido (raw deflate + base64) e o Git Bash não tem ferramenta pronta pra isso. Use a tool **PowerShell** com `System.IO.Compression.DeflateStream` — os snippets prontos estão em [references/drawio-xml-format.md](references/drawio-xml-format.md).
- **Arquivos temporários:** use o diretório de scratchpad da sessão. Se não houver um informado, use `"${TEMP}/draw-io"` (crie com `mkdir -p`).

## Instruções de execução

Ao ser invocada, verifique o conteúdo de `$ARGUMENTS`:

1. Se começar com `ler` → **fluxo de leitura**.
2. Se começar com `editar` → **fluxo de edição**.
3. Se começar com `criar` → **fluxo de criação**.
4. Caso contrário, interprete o pedido em linguagem natural e mapeie para os fluxos acima, encadeando-os quando fizer sentido (ex: "cria um diagrama X com as etapas A, B e C" = fluxo de criação seguido do fluxo de edição). Se faltar informação essencial (arquivo alvo, nome do novo arquivo), use `AskUserQuestion`.

**Regra que vale para todos os fluxos:** nunca reaproveite um `id` de `mxCell` já existente no diagrama para uma célula nova — sempre gere um `id` novo e único. Ao gravar, sempre escreva sem compressão (`compressed="false"`) — ver a seção correspondente na referência de formato.

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

Se `ACCESS_TOKEN` vier vazio, exiba o conteúdo de `$TOKEN_RESPONSE` como erro e encerre (provavelmente o refresh token expirou ou foi revogado — oriente o usuário a refazer o passo de autorização em [../google-drive/references/oauth-setup.md](../google-drive/references/oauth-setup.md)).

O access token vale para todas as chamadas do mesmo fluxo — obtenha uma vez só e reaproveite dentro do mesmo bloco/execução.

---

## Passo comum — Extrair o ID do arquivo

Analise o argumento (URL ou ID). Padrões aceitos:

- `https://drive.google.com/file/d/<ID>/view` (ou com querystring `?usp=sharing`) → `<ID>`
- `https://drive.google.com/file/d/<ID>/edit` → `<ID>`
- `https://app.diagrams.net/#G<ID>` — link de deep-link do próprio app draw.io integrado ao Drive (é comum o usuário colar este link, copiado direto do app, em vez do link do Drive) → `<ID>`
- `https://drive.google.com/drive/u/<n>/folders/<ID>` ou `https://drive.google.com/drive/folders/<ID>` → é um link de **pasta**, não de arquivo. Não trate como ID de arquivo — siga para o passo "Listar arquivos de uma pasta" abaixo.
- Se não bater com nenhum padrão de URL, trate o argumento inteiro como o próprio ID do arquivo.

Se nenhum argumento foi informado, use `AskUserQuestion` para solicitar a URL ou ID do arquivo.

---

## Passo comum — Listar arquivos de uma pasta (quando o argumento for uma pasta)

Use quando o usuário informar um link de pasta em vez de um arquivo específico (ex: pediu para ler/editar "o diagrama" e só forneceu a pasta onde ele está).

```bash
FOLDER_ID="<id extraído da URL de pasta>"

curl -s -G "https://www.googleapis.com/drive/v3/files" \
  --data-urlencode "q=\"${FOLDER_ID}\" in parents and trashed = false" \
  --data-urlencode "fields=files(id,name,mimeType,modifiedTime)" \
  --data-urlencode "pageSize=100" \
  --data-urlencode "supportsAllDrives=true" \
  --data-urlencode "includeItemsFromAllDrives=true" \
  --data-urlencode "corpora=allDrives" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "<dir temporário>/folder-list.json"
```

Leia o resultado e filtre os arquivos com `mimeType` `application/vnd.jgraph.mxfile` (ou nome terminado em `.drawio`):

- **Nenhum arquivo `.drawio` na pasta:** informe ao usuário e encerre.
- **Exatamente um:** use o `id` desse arquivo e prossiga normalmente com o fluxo pedido (leitura/edição).
- **Mais de um:** use `AskUserQuestion` listando os nomes encontrados para o usuário escolher qual arquivo.

---

## Fluxo de leitura

### Passo 1 — Extrair o ID e obter o access token

Ver passos comuns acima.

### Passo 2 — Baixar o conteúdo bruto do arquivo

```bash
FILE_ID="<id extraído>"
OUT="<dir temporário>/diagrama.drawio"

curl -s -G "https://www.googleapis.com/drive/v3/files/${FILE_ID}" \
  --data-urlencode "alt=media" \
  --data-urlencode "supportsAllDrives=true" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -o "$OUT"
```

Leia `$OUT` com a tool `Read`. Se o conteúdo for um JSON com campo `error` (em vez de XML começando com `<mxfile`), exiba a mensagem (arquivo não encontrado, sem permissão) e encerre.

### Passo 3 — Descomprimir cada página, se necessário

Para cada elemento `<diagram ...>...</diagram>` encontrado:

- Se o `<diagram>` tiver `compressed="false"`, o conteúdo entre as tags já é o XML do `mxGraphModel` — use direto.
- Caso contrário (atributo ausente ou `compressed="true"`), o conteúdo é base64+raw-deflate. Salve esse conteúdo num arquivo à parte e rode o snippet de inflate em [references/drawio-xml-format.md](references/drawio-xml-format.md) via PowerShell para obter o XML.

### Passo 4 — Parsear e reportar

Para cada página (`diagram`), liste as células (`mxCell`) do `mxGraphModel/root`, ignorando as células `id="0"` e `id="1"`:

- Vértices (`vertex="1"`): `id`, `value` (label), geometria resumida (`x,y,w,h`), style resumido (só os tokens relevantes, ex: cor, forma).
- Arestas (`edge="1"`): `id`, `source` → `target` (usando o `value` da célula de origem/destino quando disponível para ficar legível), label se houver.

Exibir no formato:

```
**<nome da página>** — <n> formas, <n> conexões

Formas:
- `node1` "Início" (retângulo, x=120,y=80)
- `node2` "Novo passo"

Conexões:
- `edge1`: "Início" → "Novo passo"
```

Se houver mais de uma página, repetir por página. Guardar o XML decodificado (por página) na sessão — o fluxo de edição reaproveita esse resultado em vez de rodar tudo de novo, se for chamado em sequência.

---

## Fluxo de edição

### Passo 1 — Ler o estado atual

Execute o **fluxo de leitura** sobre o arquivo alvo (é pré-requisito: dá o XML atual, decodificado, e os `id`s já em uso, necessários para não colidir com IDs novos).

### Passo 2 — Interpretar o pedido e planejar as mudanças

Mapeie a descrição em linguagem natural para operações sobre `mxCell`, usando o catálogo de `style` em [references/drawio-xml-format.md](references/drawio-xml-format.md):

- **Adicionar forma** → novo `mxCell vertex="1"` com `id` novo, `value`, `style` e `mxGeometry` (escolha uma posição que não sobreponha formas existentes, a menos que o pedido peça uma posição específica).
- **Adicionar conexão** → novo `mxCell edge="1"` com `source`/`target` apontando para `id`s existentes (ou para os IDs recém-criados no mesmo lote).
- **Alterar label/estilo/posição** → editar os atributos `value`/`style`/`mxGeometry` da célula existente identificada (por label, já que o usuário normalmente descreve por texto, não por `id`).
- **Remover** → apagar o `mxCell` e qualquer aresta que tenha esse `id` como `source` ou `target` (avise o usuário se isso deixar conexões "penduradas").

Se o pedido for ambíguo sobre qual célula alterar (ex: duas formas com label parecido), use `AskUserQuestion`.

### Passo 3 — Montar o XML final

Edite o XML decodificado da página alvo (texto puro, `mxGraphModel/root/...`). Escreva o resultado com a tool `Write` em um arquivo temporário, envolvendo de volta na estrutura completa:

```xml
<mxfile host="Claude Code">
  <diagram id="<mesmo id da página>" name="<mesmo nome>" compressed="false">
    <mxGraphModel ...mesmos atributos da leitura...>
      <root>
        ...células atualizadas...
      </root>
    </mxGraphModel>
  </diagram>
  ...demais páginas, sem alteração, também com compressed="false"...
</mxfile>
```

Lembrete: sempre `compressed="false"`, mesmo que o original estivesse comprimido (ver decisão de design na referência).

### Passo 4 — Gravar de volta no Drive

Atualize conteúdo **e** mimeType juntos, via `multipart`. Escreva a metadata com a tool `Write`:

```json
{"mimeType":"application/vnd.jgraph.mxfile"}
```

```bash
FILE_ID="<id do arquivo>"
METADATA="<dir temporário>/metadata-update.json"
BODY="<dir temporário>/diagrama-atualizado.drawio"

curl -s -X PATCH \
  "https://www.googleapis.com/upload/drive/v3/files/${FILE_ID}?uploadType=multipart&fields=id,name,mimeType&supportsAllDrives=true" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -F "metadata=@${METADATA};type=application/json;charset=UTF-8" \
  -F "file=@${BODY};type=application/vnd.jgraph.mxfile" \
  -o "<dir temporário>/update-response.json"
```

> **Nunca use `uploadType=media` aqui.** Nesse modo o Drive adota o `Content-Type` da requisição como mimeType do arquivo: gravar com `Content-Type: application/xml` transforma o `.drawio` em um XML genérico, que deixa de abrir no draw.io e passa a ser exibido como texto. O `multipart` acima reafirma `application/vnd.jgraph.mxfile` a cada gravação e é imune a isso.
>
> Se um arquivo já tiver sido corrompido dessa forma, o conserto é exatamente o mesmo comando — o conteúdo permanece intacto, só o mimeType precisa voltar.

### Passo 5 — Verificar e reportar

Leia a resposta. Se houver `error`, exiba `error.message` e encerre. Confirme que o campo `mimeType` retornado é `application/vnd.jgraph.mxfile` — se vier outro valor, o arquivo deixou de ser reconhecido como diagrama e a gravação precisa ser refeita pelo `multipart` do passo anterior. Em caso de sucesso, resuma o que foi alterado e informe o link:

```
Diagrama atualizado: https://drive.google.com/file/d/<FILE_ID>/view
```

---

## Fluxo de criação

### Passo 1 — Interpretar os argumentos

Do argumento, extraia:

- **nome** — texto entre aspas, ou o texto solto após `criar` (deve terminar em `.drawio`; se o usuário não incluir a extensão, adicione automaticamente).
- **pasta** — valor após `pasta:` (URL ou ID da pasta de destino); opcional.
- **descrição do conteúdo inicial** — texto após `—` (travessão) ou `-`, se houver.

Se o nome não foi informado, use `AskUserQuestion`. Se a pasta não foi informada, avise que o arquivo ficará na raiz do "Meu Drive" e siga (não bloqueie por isso).

### Passo 2 — Obter o access token

Ver passo comum acima.

### Passo 3 — Montar o XML esqueleto

Escreva com a tool `Write` (sem compressão, página única "Page-1"):

```xml
<mxfile host="Claude Code">
  <diagram id="page1" name="Page-1" compressed="false">
    <mxGraphModel dx="800" dy="600" grid="1" gridSize="10" guides="1" tooltips="1"
        connect="1" arrows="1" fold="1" page="1" pageScale="1"
        pageWidth="850" pageHeight="1100" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

### Passo 4 — Criar o arquivo no Drive (multipart)

Escreva a metadata em um arquivo separado com a tool `Write`:

```json
{"name":"<nome>.drawio","mimeType":"application/vnd.jgraph.mxfile","parents":["<folderId>"]}
```

(omita a chave `parents` se não houver pasta de destino)

```bash
METADATA="<dir temporário>/metadata.json"
CONTENT="<dir temporário>/esqueleto.drawio"

curl -s -X POST \
  "https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&fields=id,name,webViewLink&supportsAllDrives=true" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -F "metadata=@${METADATA};type=application/json;charset=UTF-8" \
  -F "file=@${CONTENT};type=application/xml" \
  -o "<dir temporário>/create-response.json"
```

Nota: se `<folderId>` (em `parents`, no metadata) for uma pasta dentro de um Drive Compartilhado, o parâmetro `supportsAllDrives=true` acima é obrigatório para a criação funcionar — sem ele a API retorna erro de permissão mesmo com o usuário tendo acesso.

Leia a resposta: `id` é o ID do novo arquivo.

### Passo 5 — Popular conteúdo inicial (se descrito) e reportar

Se o pedido original incluía descrição de conteúdo, encadeie direto para o **fluxo de edição** sobre o arquivo recém-criado (reaproveitando o XML montado no Passo 3 em vez de baixar de novo).

Ao final, informe:

```
Diagrama criado: https://drive.google.com/file/d/<FILE_ID>/view
```
