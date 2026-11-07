# Lições aprendidas

Comportamentos do GitLab CE e do ambiente Windows que **já causaram estrago em uso real**
desta skill, com a causa raiz verificada empiricamente. Leia antes de escrever na wiki.

Verificado em 2026-09-03 contra GitLab CE em `gitlab.olfar.ind.br`, Git Bash no
Windows 11.

---

## 1. `PUT` sem `title` move a subpágina para a raiz da wiki {#1}

**O que aconteceu.** Seis páginas foram atualizadas só com `content`, seguindo a regra
"nunca envie `title` num update". Cinco delas estavam em diretórios. Todas as cinco
foram movidas para a raiz da wiki: `Instalação-do-ambiente/Programas` virou `Programas`.
Os links das páginas-índice, da `home` e do `_sidebar` quebraram de uma vez.

**Causa raiz.** O GitLab recalcula o caminho do arquivo a cada `PUT`, a partir do campo
`title` da página. E `title` guarda **apenas a folha** — o diretório existe só no caminho
do arquivo, não no título. Um `PUT` sem `title` usa o título atual (`Programas`) para
montar o caminho, resultando em `Programas.md` na raiz.

Confirmado com página descartável:

| Requisição | Slug antes | Slug depois |
|---|---|---|
| `POST title=zz-teste/Pagina` | — | `zz-teste/Pagina` (mas `title` = `Pagina`) |
| `PUT content@arquivo` | `zz-teste/Pagina` | **`Pagina`** ← moveu |
| `PUT content@arquivo` + `title@arquivo` (`zz-teste/Pagina`) | `Pagina` | `zz-teste/Pagina` ← voltou |

**Regra.** Ao atualizar uma página cujo slug contém `/`, envie **sempre** o `title` com o
caminho completo. Só omita `title` em páginas da raiz.

**Como reverter se acontecer.** Mover de volta pela API exige um `PUT` com o `title`
completo por página. Se forem muitas, o repositório git da wiki é mais rápido e atômico
(ver §4). Confira depois os redirecionamentos (§3).

---

## 2. Acento em argumento de linha de comando → HTTP 400 {#2}

**O que aconteceu.** `--data-urlencode "title=Arquitetura/Integrações de saída"` devolveu
`400 Bad Request` — resposta em `text/plain`, 11 bytes, sem corpo JSON. A mesma chamada
com título sem acento devolvia `200`.

**Causa raiz.** O `curl` do Git Bash é um **binário nativo do Windows**. O MSYS2
transcodifica os argumentos da linha de comando para a codepage ANSI ao invocá-lo, então
`á` (`C3 A1` em UTF-8) chega ao curl como `E1` — byte inválido em UTF-8. O Rack rejeita a
requisição antes de o GitLab vê-la, daí a resposta em texto puro em vez de JSON.

O engano é fácil de cometer porque **os bytes no shell estão corretos**: `printf '%s' "…"
| od` mostra `C3 A1`. A corrupção acontece só na fronteira com o processo nativo.

| Forma | Bytes no bash | Resultado |
|---|---|---|
| `--data-urlencode "title=Página Acentuada"` | `C3 A1` | **400 Bad Request** |
| `--data-urlencode "title@arquivo.txt"` | `C3 A1` | `200 OK` |

**Regra.** Nenhum valor com caractere não-ASCII vai pela linha de comando. Grave num
arquivo com a tool `Write` (que escreve UTF-8) e use `campo@arquivo`. Vale para `title`,
e para qualquer parâmetro futuro com texto livre.

Isso **não** afeta o caminho da URL: ali os acentos já vão percent-encodados por você, em
ASCII puro.

**Sintoma para reconhecer.** `400` com corpo `Bad Request` em texto puro (o GitLab
sempre responde erro em JSON) e `X-Runtime` na casa dos milissegundos = a requisição
morreu antes da aplicação, quase sempre por encoding.

---

## 3. O GitLab **cria** redirecionamentos — e eles podem virar lixo {#3}

Renomear ou mover uma página grava uma entrada `origem: destino` em
`.gitlab/redirects.yml`, no repositório da wiki. Links antigos continuam funcionando.

**O risco.** O arquivo é um mapa acumulativo que o GitLab só sabe **adicionar**. Uma
movimentação errada seguida da correção deixa entradas apontando para páginas que já não
existem — e apaga as entradas legítimas no sentido inverso. Foi o que aconteceu no
episódio da §1: os cinco caminhos corretos passaram a redirecionar para os caminhos na
raiz, que tinham deixado de existir.

**Regra.** Depois de qualquer renomeação ou movimentação, leia o arquivo e confira que
todo destino existe na listagem de páginas. Entradas inválidas se limpam por commit no
repositório da wiki (§4).

O arquivo também acumula lixo de erros humanos antigos — já foi encontrada uma entrada com
o literal `{new_page_title}`, de uma renomeação malfeita pela interface. Não tente
"consertar" o histórico: remova só o que você mesmo criou, e pergunte antes de mexer no
resto.

---

## 4. O repositório git da wiki é a saída para o que a API não faz bem {#4}

A wiki é um repositório git (`<url-do-projeto>.wiki.git`) e aceita push. Clonar é a
ferramenta certa para três coisas:

```bash
git clone "https://oauth2:${GITLAB_TOKEN}@${GITLAB_URL#https://}/<grupo/projeto>.wiki.git" "<scratchpad>/wikiclone"
```

**a) Backup.** Bytes idênticos aos do servidor, com anexos (`uploads/`) e
`.gitlab/redirects.yml` — coisas que a API de wikis não devolve. Muito melhor que
reconstruir arquivos a partir dos escapes JSON de `?with_content=1`.

**b) Movimentações em lote.** `git mv` + commit + push move páginas entre diretórios
preservando o conteúdo, num único commit, sem o risco da §1.

**c) Limpeza de `.gitlab/redirects.yml`**, que a API REST não expõe.

Cuidados ao usar o clone:

- Rode `git config core.quotepath false`, senão `git ls-files` devolve os nomes acentuados
  escapados em octal e seus `grep` não casam.
- Confira `core.autocrlf` antes de confiar num backup: se estiver `true`, o checkout
  converte os finais de linha e o backup deixa de ser byte-exato. Valide contando os `\r`
  de cada arquivo contra o que a API devolve.
- Nunca imprima a URL do remote — ela contém o token. Filtre a saída com
  `sed "s|${GITLAB_TOKEN}|***|g"`.

---

## 5. Faça backup antes de qualquer operação em lote {#5}

O histórico git da wiki já permite recuperar tudo, mas depender dele no meio de um
incidente é lento. Um clone em `BKP/<AAAAMMDD-HHMM>/` custa segundos e transforma a
recuperação num `cp` mais um push.

No episódio da §1, foi o backup que permitiu restaurar `.gitlab/redirects.yml` ao estado
exato anterior — algo que nenhum comando do GitLab reconstrói.

Depois de terminar, **compare o resultado com o backup** arquivo por arquivo. As únicas
diferenças aceitáveis são as que você pretendia fazer; qualquer outra é conteúdo perdido.

---

## 6. Cuidado ao escrever comandos: este ambiente tem dois shells {#6}

A sessão expõe PowerShell **e** Bash, com sintaxes incompatíveis. Um here-string do
PowerShell (`@'…'@`) passado ao Bash não dá erro — ele vira argumento literal e contamina
o resultado silenciosamente. Já produziu uma mensagem de commit com `@` na primeira linha.

No Bash use here-doc (`<<'EOF'`); no PowerShell use `@'…'@`. E confira o resultado de
comandos que gravam texto, em vez de assumir que funcionou porque o exit code foi `0`.
