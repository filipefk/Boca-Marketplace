# Regras de escrita na wiki do GitLab

## Slug × título

O slug é derivado do título no momento em que a página é criada ou renomeada:

| Título | Slug resultante | Na URL da API |
|---|---|---|
| `Home` | `Home` | `Home` |
| `Instalação do ambiente` | `Instalação-do-ambiente` | `Instala%C3%A7%C3%A3o-do-ambiente` |
| `Deploy/Rollback` | `Deploy/Rollback` | `Deploy%2FRollback` |
| `CI CD` | `CI-CD` | `CI-CD` |

Regras práticas:

- Espaço vira hífen; acentos e maiúsculas são **preservados** no slug.
- **Nunca monte o slug na mão para uma página existente** — pegue o campo `slug` devolvido por `GET /wikis` e apenas percent-encode. Adivinhar dá `404`.
- Ao **criar**, prefira títulos sem acento quando o slug for virar link em outros lugares: `Instalacao-do-ambiente` é mais fácil de colar e citar do que `Instala%C3%A7%C3%A3o-do-ambiente`. Se o usuário quiser o título acentuado, respeite a escolha dele — só avise.
- Não use `:` `?` `#` `[` `]` no título; a wiki aceita, mas o slug fica hostil na URL.

## Diretórios

Barra no título cria diretório: `Ambiente/Instalação` põe a página `Instalação` na pasta `Ambiente`. Isso é a única forma de organizar a wiki em níveis — não existe "mover para pasta" separado; mover = renomear com o novo caminho.

Um diretório existe enquanto tiver páginas dentro. Apagar a última página apaga o diretório.

**O diretório não fica guardado no título.** Depois de criar `Ambiente/Instalação`, a API devolve `slug: "Ambiente/Instalação"` mas `title: "Instalação"` — só a folha. Como o GitLab recalcula o caminho do arquivo a partir do `title` a cada `PUT`, **um update sem `title` joga a página para a raiz da wiki**. Ao atualizar qualquer página cujo slug tenha `/`, envie sempre o `title` com o caminho completo. Ver [licoes-aprendidas.md](licoes-aprendidas.md#1).

## Páginas especiais

| Slug | Papel |
|---|---|
| `home` | Página inicial da wiki — é o que abre em `/-/wikis/home` |
| `_sidebar` | Menu lateral customizado (substitui a árvore automática) |

Ao criar a primeira página de uma wiki vazia, sugira ao usuário que ela seja a `home`, ou que a `home` seja criada com um índice apontando para as demais.

**Se a wiki tiver `_sidebar`, ele substitui a árvore automática do GitLab.** Toda página nova precisa ser acrescentada a ele à mão, senão fica invisível na navegação lateral, mesmo existindo. Ao criar uma página numa wiki com `_sidebar`, atualize-o no mesmo trabalho — e avise o usuário se ele preferir deixar para depois.

## Conteúdo

O GitLab renderiza Markdown (GFM) no campo `content`. Convenções para páginas geradas por esta skill:

- Comece com um `#` de título igual ao título da página, seguido de uma frase de uma linha dizendo para que serve a página.
- Use `##` para as seções principais; evite passar de `###`.
- Blocos de código sempre com a linguagem no fence (```bash, ```php, ```sql) — a wiki faz syntax highlight.
- Tabelas para inventários (versões, variáveis, caminhos); listas numeradas para passo a passo.
- Links entre páginas da wiki são relativos ao slug: `[Instalação](Instalacao-do-ambiente)`.
- Não coloque no rodapé assinatura de ferramenta ou data de geração — o histórico de commits da wiki já registra autor e data.

## Encoding

A API trabalha em **UTF-8**. Antes de publicar um arquivo local, confira o encoding — repositórios legados frequentemente têm arquivos em ISO-8859-1 (latin-1), que sobem com acentos corrompidos.

Detectar (heurística: acento em latin-1 aparece como byte isolado inválido em UTF-8):

```bash
file -i caminho/arquivo.md
```

Converter para um arquivo temporário antes de enviar (nunca converta o arquivo original no repositório do usuário sem pedir):

```bash
iconv -f ISO-8859-1 -t UTF-8 caminho/arquivo.md > "<scratchpad>/wiki-content.md"
```

Se `iconv` não existir, leia o arquivo com a tool `Read` e regrave com a tool `Write`, que escreve em UTF-8.

## Backup antes de operação em lote

Se a tarefa vai escrever em mais de uma ou duas páginas — normalização, reorganização,
publicação de várias páginas de uma vez — faça um backup antes e conte ao usuário onde
ele ficou. Clonar o repositório da wiki leva segundos e captura o que a API não devolve
(anexos e `.gitlab/redirects.yml`):

```bash
git clone "https://oauth2:${GITLAB_TOKEN}@${GITLAB_URL#https://}/<grupo/projeto>.wiki.git" "BKP/<AAAAMMDD-HHMM>"
```

Ao terminar, compare o resultado com o backup arquivo por arquivo: as únicas diferenças
aceitáveis são as pretendidas. Ver [licoes-aprendidas.md](licoes-aprendidas.md#5).

## Confirmação antes de escrever

Sempre use `AskUserQuestion` antes de qualquer chamada que altere a wiki — criar, atualizar, renomear, apagar, anexar. Se o usuário cancelar, encerre sem chamar a API.

Motivo: a wiki não tem rascunho, revisão nem merge request. O `PUT` publica na hora, para todos os membros do projeto, e sobrescreve a página inteira.

Na confirmação, exiba:

```
Projeto: <path_with_namespace>
Página:  <título>  (slug: <slug>)
Ação:    Criar | Atualizar (sobrescreve) | Renomear | Apagar | Anexar
Origem:  <arquivo local, se houver>
Formato: markdown

Prévia:
<primeiras ~15 linhas do conteúdo>
```

Para atualização, prefira mostrar o que muda (seções adicionadas/removidas/alteradas) em vez de despejar o conteúdo inteiro.

## Antes de publicar, verifique o conteúdo

Publicar na wiki é publicar. Antes de enviar um arquivo local, leia-o e recuse-se a subir — avisando o usuário — se contiver senha, token, string de conexão, chave privada, dados pessoais ou caminho com credencial embutida. Ofereça publicar uma versão com os valores substituídos por placeholders.
