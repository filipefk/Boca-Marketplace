---
name: gitlab-wiki
description: Lista, lê, cria, atualiza, publica a partir de arquivo local, exporta, apaga e anexa arquivos em páginas da Wiki de um projeto do GitLab (Community Edition, API REST v4). Projeto e página são resolvidos por nome, slug ou URL. Usa as mesmas credenciais da skill gitlab-card — GITLAB_URL e GITLAB_TOKEN (e opcionalmente GITLAB_PROJECT).
---

## Como usar

```
/gitlab-wiki listar
/gitlab-wiki listar projeto:"grupo/projeto"
/gitlab-wiki ler <slug | URL da página>
/gitlab-wiki criar titulo:"Instalação do ambiente" <conteúdo ou instrução do que escrever>
/gitlab-wiki criar titulo:"Deploy/Rollback" arquivo:"docs/rollback.md"
/gitlab-wiki publicar <caminho do arquivo local> [titulo:"..."]
/gitlab-wiki atualizar <slug> <novo conteúdo ou instrução>
/gitlab-wiki atualizar <slug> arquivo:"INSTALACAO_AMBIENTE.md"
/gitlab-wiki renomear <slug> titulo:"Novo título"
/gitlab-wiki exportar <slug> [destino:"caminho/arquivo.md"]
/gitlab-wiki apagar <slug>
/gitlab-wiki anexar <caminho do arquivo local>
```

## Referências

- [Configuração do token](references/token-setup.md) — variáveis de ambiente e permissões (as mesmas da skill `gitlab-card`)
- [Regras da wiki](references/wiki-rules.md) — slug × título, diretórios, encoding, sidebar, confirmação
- [Referência da API v4](references/api-reference.md) — endpoints, campos, limitações e códigos de erro
- **[Lições aprendidas](references/licoes-aprendidas.md)** — armadilhas que já causaram estrago em uso real: `PUT` que move subpáginas para a raiz, acento em argumento que devolve `400`, redirecionamentos que viram lixo. **Leia antes da primeira escrita numa sessão.**

## Variáveis de ambiente

| Variável         | Obrigatória | Descrição                                                                    |
|------------------|-------------|------------------------------------------------------------------------------|
| `GITLAB_URL`     | sim         | URL base da instância, sem barra final (ex.: `https://gitlab.olfar.ind.br`) |
| `GITLAB_TOKEN`   | sim         | Personal Access Token com escopo `api`                                       |
| `GITLAB_PROJECT` | não         | Projeto padrão: caminho `grupo/projeto` ou id numérico                        |

São exatamente as mesmas variáveis usadas pela skill `gitlab-card` — se ela já funciona, esta também funciona, sem configuração adicional.

Valide no início de cada fluxo:

```bash
GITLAB_URL="${GITLAB_URL:?Variável GITLAB_URL não encontrada — veja references/token-setup.md}"
GITLAB_TOKEN="${GITLAB_TOKEN:?Variável GITLAB_TOKEN não encontrada — veja references/token-setup.md}"
API="${GITLAB_URL%/}/api/v4"
```

Toda chamada é autenticada pelo header `PRIVATE-TOKEN`. **Nunca imprima o valor de `GITLAB_TOKEN`** nem o inclua na URL — deixe sempre o shell expandir a variável dentro do `curl`.

## Conceitos da wiki do GitLab

- A wiki é um **repositório git separado** do projeto (`<url-do-projeto>.wiki.git`). Cada página é um arquivo (`.md` por padrão) e cada escrita pela API é um commit.
- A página é identificada pelo **`slug`**, não por id. O slug vem do título: espaços viram hífen, e o texto é preservado (inclusive acentos).
- **Diretórios existem**: um título `Deploy/Rollback` cria a página `Rollback` dentro da pasta `Deploy`, com slug `Deploy/Rollback`. Na URL da API a barra vira `%2F`.
- A página inicial tem slug **`home`**. A página `_sidebar` controla o menu lateral.
- **Não há rascunho, revisão nem MR**: `POST`/`PUT`/`DELETE` alteram a wiki imediatamente para todos.
- **Toda página fica onde está, a menos que o usuário peça explicitamente para movê-la.** Atualizar conteúdo (`atualizar`, `publicar`) nunca pode mudar o diretório nem o slug: `Pipelines/Pipeline-de-deploy` continua abaixo de `Pipelines` depois do update. Só o fluxo de renomeação/movimentação, acionado por pedido explícito, muda o local. Como o `PUT` do GitLab tende a jogar subpáginas para a raiz, isso exige enviar o `title` com o caminho completo e conferir o slug depois (ver fluxo de atualização).
- **Wiki de grupo** (`/groups/:id/wikis`) tem os mesmos endpoints, mas é recurso **Premium/Ultimate** — não existe no CE. Se o usuário pedir wiki de grupo em uma instância CE, avise e ofereça a wiki do projeto.

## Instruções de execução

Ao ser invocada, verifique `$ARGUMENTS` na ordem abaixo (a primeira regra que casar vence):

1. Começa com `listar` → **fluxo de listagem**.
2. Começa com `ler`, ou contém uma URL no formato `<GITLAB_URL>/<caminho>/-/wikis/<slug>` → **fluxo de leitura**.
3. Começa com `criar` → **fluxo de criação**.
4. Começa com `publicar` → **fluxo de publicação de arquivo local**.
5. Começa com `atualizar` → **fluxo de atualização**.
6. Começa com `renomear` → **fluxo de renomeação/movimentação**.
7. Começa com `exportar` → **fluxo de exportação**.
8. Começa com `apagar` (ou `excluir`, `deletar`) → **fluxo de exclusão**.
9. Começa com `anexar` → **fluxo de anexo**.
10. Nenhum verbo reconhecido → pergunte com `AskUserQuestion` o que fazer, listando as opções mais prováveis para o texto informado.

---

## Passo comum — Resolver projeto e página

### A. Projeto → `PROJECT_ID`

Ordem de precedência:

1. Caminho extraído da URL da página (o trecho entre o host e `/-/wikis/`), se houver.
2. Valor do parâmetro `projeto:"..."`.
3. Conteúdo de `$GITLAB_PROJECT`.
4. Nada informado → liste os projetos e pergunte com `AskUserQuestion` (máximo 4 opções; se houver mais, mostre os 4 primeiros e avise que há outros).

Se o valor contiver `/`, é um caminho: use-o direto como `:id`, **URL-encoded** (`grupo/projeto` → `grupo%2Fprojeto`; subgrupos codificam cada barra). Se for numérico, use-o direto. Caso contrário, busque pelo nome:

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects?membership=true&simple=true&per_page=100&search=<nome>"
```

Case `name` ou `path_with_namespace` (case-insensitive). Zero resultados → erro e encerre. Mais de um → `AskUserQuestion`. Guarde `id` como `PROJECT_ID` e o `web_url`.

### B. Página → `SLUG` e `SLUG_ENC`

Aceite três formas:

- **Slug direto** (`home`, `Deploy/Rollback`).
- **URL** `https://gitlab.olfar.ind.br/grupo/projeto/-/wikis/Deploy/Rollback` → tudo depois de `/-/wikis/` é o slug; o trecho antes é o projeto (tem precedência).
- **Título aproximado** ("instalação do ambiente") → liste as páginas (fluxo de listagem, Passo 2) e case por `title` ou `slug`, case-insensitive e ignorando hífens. Mais de um candidato → `AskUserQuestion`. Nenhum → informe e ofereça criar.

**Sempre prefira o `slug` devolvido pela API** a montar um slug na mão. Para usar no caminho da URL, gere `SLUG_ENC` percent-encodando: `/` → `%2F`, espaço → `%20`, e caracteres acentuados na forma percent-encoded UTF-8. Detalhes e exemplos em [wiki-rules.md](references/wiki-rules.md).

### C. Conteúdo e título → arquivos temporários

Todo valor enviado à API sai de um **arquivo em disco**, nunca de uma string inline:

- `arquivo:"caminho"` informado → use o arquivo indicado (confira o encoding, ver [wiki-rules.md](references/wiki-rules.md)).
- Conteúdo redigido por você → grave com a tool `Write` (UTF-8) em `<scratchpad>/wiki-content.md`.
- **Título** (em criação, atualização de subpágina ou renomeação) → grave em
  `<scratchpad>/wiki-title.txt`, sem quebra de linha no fim.

Isso resolve dois problemas de uma vez: escapar Markdown multilinha dentro de JSON, e a
corrupção de acentos na linha de comando do Git Bash — que devolve `400 Bad Request` e é
fácil de diagnosticar errado, porque os bytes no shell estão corretos. Ver
[licoes-aprendidas.md](references/licoes-aprendidas.md#2).

---

## Fluxo de listagem

### Passo 1 — Resolver o projeto

Execute a parte **A** do Passo comum.

### Passo 2 — Listar as páginas

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/wikis" \
  -o "<scratchpad>/wikis.json"
```

Leia o arquivo com a tool `Read`. **Não** use `with_content=1` numa listagem só para exibir os títulos — traz o conteúdo inteiro de todas as páginas.

### Passo 3 — Exibir

Agrupe por diretório (o trecho antes da última `/` do slug) e ordene alfabeticamente:

```
Wiki de <path_with_namespace> — <N> páginas

  home
  Instalação-do-ambiente
  Deploy/
    Deploy/Rollback
```

Se a resposta vier vazia (`[]`), informe que a wiki não tem nenhuma página ainda. Se vier `404`, veja a tabela de erros em [api-reference.md](references/api-reference.md) — provavelmente a wiki está desabilitada no projeto.

---

## Fluxo de leitura

### Passo 1 — Resolver projeto e página

Execute as partes **A** e **B** do Passo comum.

### Passo 2 — Buscar a página

```bash
curl -s -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${API}/projects/${PROJECT_ID}/wikis/${SLUG_ENC}" \
  -o "<scratchpad>/wiki-page.json"
```

Leia com a tool `Read`. O campo `content` vem como string JSON escapada (`\n`, `\"`) — ao exibir, converta os escapes de volta.

### Passo 3 — Exibir

```
Página:  <title>
Slug:    <slug>
Projeto: <path_with_namespace>
Formato: <format>
URL:     <GITLAB_URL>/<path_with_namespace>/-/wikis/<slug>

<conteúdo em Markdown>
```

Se o conteúdo for longo, exiba as primeiras ~80 linhas e ofereça `exportar` para gravar o arquivo completo em disco.

---

## Fluxo de criação

### Passo 1 — Extrair parâmetros

De `$ARGUMENTS` (após remover `criar`), extraia:

- `titulo:"..."` — título da página (obrigatório; se ausente, infira do conteúdo e confirme no Passo 3)
- `arquivo:"..."` — caminho de um arquivo local a usar como conteúdo
- `formato:"..."` — `markdown` (padrão), `rdoc`, `asciidoc` ou `org`
- `projeto:"..."` — projeto de destino
- O restante do texto é o conteúdo, ou a instrução do que escrever

### Passo 2 — Resolver projeto e conteúdo

Execute as partes **A** e **C** do Passo comum.

Se o usuário descreveu o que quer em vez de fornecer o texto pronto, redija o Markdown seguindo as convenções de [wiki-rules.md](references/wiki-rules.md) e grave no arquivo temporário.

### Passo 3 — Verificar se a página já existe e confirmar

Liste as páginas (fluxo de listagem, Passo 2) e verifique se já existe uma com o slug equivalente ao título.

- **Já existe** → não crie duplicata: pergunte com `AskUserQuestion` entre **Atualizar a página existente** (segue para o fluxo de atualização), **Criar com outro título** e **Cancelar**.
- **Não existe** → exiba título, slug previsto, projeto, formato e as primeiras linhas do conteúdo, e confirme com `AskUserQuestion`: **Criar página** ou **Cancelar**.

### Passo 4 — Criar via API

Grave o título em `<scratchpad>/wiki-title.txt` (UTF-8, com a tool `Write`, **sem quebra de linha no fim**) e envie os dois campos por arquivo:

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/wikis" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --data-urlencode "title@<scratchpad>/wiki-title.txt" \
  --data-urlencode "format=markdown" \
  --data-urlencode "content@<scratchpad>/wiki-content.md" \
  -w "\n%{http_code}\n"
```

`--data-urlencode "campo@arquivo"` lê o arquivo e o codifica — é o que permite enviar texto acentuado e Markdown multilinha sem escapar nada, **e sem passar pela linha de comando**, onde acentos são corrompidos no Windows (ver [licoes-aprendidas.md](references/licoes-aprendidas.md#2)).

Para criar a página **dentro de um diretório**, o título é o caminho completo: `Arquitetura/Integrações de saída` gera o slug `Arquitetura/Integrações-de-saída`. Note que o GitLab guarda em `title` só a folha (`Integrações de saída`) — o diretório existe apenas no caminho do arquivo.

### Passo 5 — Reportar

Sucesso (`201`): exiba título, `slug` devolvido e a URL `<GITLAB_URL>/<path_with_namespace>/-/wikis/<slug>`. Erro: exiba o `message`/`error` e consulte a tabela em [api-reference.md](references/api-reference.md).

---

## Fluxo de publicação de arquivo local

Atalho para publicar um arquivo `.md` do repositório na wiki: cria a página se não existir, atualiza se existir.

### Passo 1 — Resolver o arquivo

O primeiro argumento após `publicar` é o caminho do arquivo (relativo ao diretório de trabalho ou absoluto). Confirme que existe e **verifique o encoding** — arquivos em ISO-8859-1 precisam ser convertidos para UTF-8 antes do envio (ver [wiki-rules.md](references/wiki-rules.md)).

### Passo 2 — Determinar o título

Nesta ordem: parâmetro `titulo:"..."` → primeiro heading `#` do arquivo → nome do arquivo sem extensão, com `_` e `-` virando espaço.

### Passo 3 — Decidir entre criar e atualizar

Execute a parte **A** do Passo comum e liste as páginas. Se já existir página com o slug equivalente — **inclusive dentro de um diretório** (compare também com a folha do slug: o arquivo `PIPELINE_DE_DEPLOY.md` casa com `Pipelines/Pipeline-de-deploy`) —, siga o fluxo de atualização (mostrando na confirmação que é uma sobrescrita). Caso contrário, siga o fluxo de criação a partir do Passo 3.

Ao atualizar, **o local da página existente prevalece** sobre o título derivado do arquivo: o `title` enviado é o caminho do slug atual, não o nome do arquivo local. Publicar nunca move a página.

Ao confirmar, deixe explícito **qual arquivo local** vai para **qual página**, e que a wiki é pública para todos os membros do projeto.

---

## Fluxo de atualização

Substitui o conteúdo de uma página existente. **A escrita sobrescreve a página inteira** — não há merge.

### Passo 1 — Resolver projeto, página e conteúdo

Execute as partes **A**, **B** e **C** do Passo comum.

### Passo 2 — Ler o conteúdo atual

Execute o fluxo de leitura (Passo 2). Se o usuário pediu uma alteração pontual ("acrescente uma seção X", "corrija o passo 3"), parta do conteúdo atual, aplique a mudança e grave o texto **completo** no arquivo temporário — enviar só o trecho novo apagaria o resto.

### Passo 3 — Confirmar

Exiba o slug, o resumo do que muda (idealmente um diff resumido: seções adicionadas, removidas, alteradas) e confirme com `AskUserQuestion`: **Atualizar página** ou **Cancelar**.

### Passo 4 — Atualizar via API

**Página na raiz da wiki** (slug sem `/`):

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/wikis/${SLUG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --data-urlencode "content@<scratchpad>/wiki-content.md" \
  -w "\n%{http_code}\n"
```

**Subpágina** (slug com `/`) — envie **também** o `title` com o caminho completo, **derivado do slug atual devolvido pela API** (nunca do título informado pelo usuário nem do nome de um arquivo local). Ex.: slug `Pipelines/Pipeline-de-deploy` → `wiki-title.txt` contém `Pipelines/Pipeline-de-deploy`.

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/wikis/${SLUG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --data-urlencode "content@<scratchpad>/wiki-content.md" \
  --data-urlencode "title@<scratchpad>/wiki-title.txt" \
  -w "\n%{http_code}\n"
```

> ⚠️ **Omitir o `title` numa subpágina move a página para a raiz da wiki.** O campo `title` guarda só a folha (`Programas`), não o caminho — e o GitLab recalcula o caminho do arquivo a partir dele a cada `PUT`. Sem `title`, `Instalação-do-ambiente/Programas` vira `Programas`, quebrando todos os links. Grave em `wiki-title.txt` o caminho completo com a folha em forma de título: `Instalação-do-ambiente/Programas`. Ver [licoes-aprendidas.md](references/licoes-aprendidas.md#1).
>
> ⚠️ Passe o título **por arquivo** (`title@arquivo`), nunca inline (`title=...`), sempre que ele tiver acento. Ver [licoes-aprendidas.md](references/licoes-aprendidas.md#2).

Um `title` **diferente** do título atual renomeia a página e muda o slug. Para renomear de propósito, use o fluxo próprio.

### Passo 5 — Conferir que a página não saiu do lugar

Compare o `slug` da resposta do `PUT` com o `SLUG` original. **Têm que ser idênticos.**

Se forem diferentes (ex.: `Pipelines/Pipeline-de-deploy` virou `Pipeline-de-deploy`), a página foi movida sem o usuário pedir: avise imediatamente e devolva-a ao lugar original com um `PUT` no **slug novo** enviando o `title` com o caminho original (`title@wiki-title.txt` contendo `Pipelines/Pipeline-de-deploy`). Confira de novo o slug devolvido e verifique o `.gitlab/redirects.yml` (fluxo de renomeação, Passo 3), porque o vaivém pode deixar entradas de redirecionamento sobrando. Ver [licoes-aprendidas.md](references/licoes-aprendidas.md#1).

### Passo 6 — Reportar

Exiba o slug (confirmando que é o mesmo de antes), a URL da página e o que foi alterado.

---

## Fluxo de renomeação/movimentação

Renomear é enviar um `title` novo; mover para outro diretório é enviar um título com `/`.

### Passo 1 — Resolver e avisar

Execute as partes **A** e **B** do Passo comum. Exiba o slug atual e o slug resultante.

O GitLab **registra um redirecionamento** do slug antigo para o novo em `.gitlab/redirects.yml`, no repositório da wiki — então links antigos continuam funcionando. Ainda assim, avise: o redirecionamento é um mapa acumulativo e frágil (uma renomeação malfeita deixa entradas apontando para páginas inexistentes), e links em sistemas externos ficam obsoletos.

Confirme com `AskUserQuestion`: **Renomear** ou **Cancelar**.

### Passo 2 — Renomear via API

Grave o novo título em `<scratchpad>/wiki-title.txt` (UTF-8, com a tool `Write`) e envie por arquivo:

```bash
curl -s -X PUT "${API}/projects/${PROJECT_ID}/wikis/${SLUG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --data-urlencode "title@<scratchpad>/wiki-title.txt" \
  -w "\n%{http_code}\n"
```

Para **mover para outro diretório**, o título é o caminho completo: `Pipelines/Pipeline de deploy`. Para mover para a **raiz**, um título sem `/`.

Reporte o slug antigo, o novo e a URL.

### Passo 3 — Conferir os redirecionamentos

Depois de qualquer renomeação ou movimentação, verifique se o mapa de redirecionamentos ficou coerente — nenhuma entrada pode apontar para uma página que não existe. O arquivo fica no **repositório da wiki**, que a API REST não expõe; leia-o pelo git:

```bash
git clone --quiet "https://oauth2:${GITLAB_TOKEN}@${GITLAB_URL#https://}/<grupo/projeto>.wiki.git" "<scratchpad>/wikiclone"
cat "<scratchpad>/wikiclone/.gitlab/redirects.yml"
```

Cada linha é `origem: destino`. Confira que todo destino existe na listagem de páginas. Se houver entradas inválidas, avise o usuário e ofereça limpá-las por commit no repositório da wiki (ver [licoes-aprendidas.md](references/licoes-aprendidas.md#4)).

---

## Fluxo de exportação

Grava o conteúdo da página em um arquivo local.

### Passo 1 — Resolver projeto e página

Execute as partes **A** e **B** do Passo comum e busque a página (fluxo de leitura, Passo 2).

### Passo 2 — Gravar o arquivo

Destino: parâmetro `destino:"..."`, ou `<slug com / virando _>.md` no diretório de trabalho atual. Se o arquivo já existir, confirme a sobrescrita com `AskUserQuestion`.

Leia o JSON com a tool `Read`, converta os escapes (`\n`, `\"`, `\\`) e grave o Markdown com a tool `Write` (UTF-8).

Para páginas muito grandes — ou para exportar a wiki inteira — clonar o repositório da wiki é mais eficiente que a API; ver [api-reference.md](references/api-reference.md).

### Passo 3 — Reportar

Informe o caminho gravado e o tamanho.

---

## Fluxo de exclusão

### Passo 1 — Resolver e confirmar

Execute as partes **A** e **B** do Passo comum. **Leia a página antes** (fluxo de leitura) e exiba título, slug e as primeiras linhas do conteúdo — nunca apague às cegas.

Ofereça exportar antes de apagar. Confirme com `AskUserQuestion`: **Apagar página** ou **Cancelar**.

### Passo 2 — Apagar via API

```bash
curl -s -X DELETE "${API}/projects/${PROJECT_ID}/wikis/${SLUG_ENC}" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  -w "\n%{http_code}\n"
```

Sucesso é `204` com corpo vazio. Reporte o que foi apagado e lembre que o histórico continua no repositório git da wiki (a página pode ser recuperada por um commit anterior).

---

## Fluxo de anexo

Sobe um arquivo para a pasta `uploads/` da wiki e devolve o Markdown para referenciá-lo.

### Passo 1 — Resolver projeto e arquivo

Execute a parte **A** do Passo comum. Confirme que o arquivo existe e informe nome e tamanho na confirmação (`AskUserQuestion`: **Anexar** ou **Cancelar**).

### Passo 2 — Subir via API

```bash
curl -s -X POST "${API}/projects/${PROJECT_ID}/wikis/attachments" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --form "file=@<caminho do arquivo>" \
  -w "\n%{http_code}\n"
```

### Passo 3 — Reportar

Exiba `file_path` e o `link.markdown` devolvido, prontos para colar numa página. Se o usuário indicou uma página de destino, ofereça acrescentar o Markdown a ela pelo fluxo de atualização.
