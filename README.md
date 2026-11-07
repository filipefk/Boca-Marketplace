# Mercado do Boca - Plugins para Claude Code

Marketplace de plugins do Claude Code.

---

## O que é um Marketplace de plugins do Claude Code?

O Claude Code possui um sistema de **plugins** que permite estender suas capacidades com novas skills (habilidades). Um **Marketplace** é um repositório Git (ou pasta local) que empacota um conjunto de plugins prontos para uso, tornando fácil compartilhá-los e instalá-los com um único comando.

Cada plugin é composto por arquivos `SKILL.md` — documentos em Markdown que ensinam o Claude Code a executar uma tarefa específica (chamadas de API, processamento de texto, interação com ferramentas externas), tudo orquestrado pelo próprio Claude.

---

## Como instalar no Claude Code

### Pré-requisitos

1. Git instalado na máquina: https://git-scm.com/install/
2. Claude Code CLI instalado e configurado.

### Instalação

Abra o Claude Code CLI e adicione o marketplace apontando para o repositório Git ou para a pasta local:

`/plugin marketplace add <owner/repositorio ou caminho da pasta>`

Com o marketplace adicionado, instale os plugins desejados:

```
/plugin install google-plugin@boca-marketplace
/plugin install gitlab-plugin@boca-marketplace
/plugin install azure-plugin@boca-marketplace
/plugin install board-plugin@boca-marketplace
```

O marketplace e os plugins também aparecem na interface do Claude Code Desktop e podem ser gerenciados por lá (instalar, desabilitar, remover...).

## Atualização automática

O Claude Code pode atualizar automaticamente marketplaces e seus plugins na inicialização. Veja: [Configure atualizações automáticas](https://code.claude.com/docs/pt/discover-plugins#configure-auto-updates)

---

## Plugins disponíveis

### `google-plugin` — Google Drive, Docs, Sheets, Slides e draw.io

| Skill | O que faz |
|---|---|
| `google-drive` | Lista o conteúdo de uma pasta do Drive a partir de URL ou ID |
| `google-docs` | Lê, cria (em branco ou a partir de template) e edita documentos |
| `google-sheets` | Lê, escreve, formata e cria planilhas e abas |
| `google-slides` | Inspeciona, cria (em branco ou a partir de template) e preenche apresentações |
| `draw-io` | Lê, edita e cria diagramas `.drawio` (diagrams.net) armazenados no Drive, manipulando o XML do mxGraph |

**Variáveis de ambiente:** `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN` (compartilhadas por todas as skills; Docs, Sheets, Slides e draw-io exigem escopo de escrita).

Passo a passo para criar a credencial OAuth e gerar o refresh token: [oauth-setup.md](plugins/google-plugin/skills/google-drive/references/oauth-setup.md).

---

### `gitlab-plugin` — GitLab (Community Edition, API REST v4)

| Skill | O que faz |
|---|---|
| `gitlab-card` | Cria, lê, atualiza, move, fecha, reabre, comenta e lista issues no board; gerencia checklists |
| `gitlab-pr` | Lista, lê, revisa e comenta merge requests; responde e resolve threads; cria MR. Não faz merge nem aprova |
| `gitlab-release` | Lista, lê, cria, atualiza e apaga releases; gera notas da release a partir das mudanças desde a tag anterior |
| `gitlab-wiki` | Lista, lê, cria, atualiza, publica, exporta, apaga e anexa arquivos em páginas da Wiki |

**Variáveis de ambiente:** `GITLAB_URL`, `GITLAB_TOKEN` e, opcionalmente, `GITLAB_PROJECT` (compartilhadas por todas as skills).

Como gerar o Personal Access Token e configurar as variáveis: [token-setup.md](plugins/gitlab-plugin/skills/gitlab-card/references/token-setup.md).

---

### `azure-plugin` — Azure DevOps

| Skill | O que faz |
|---|---|
| `azure-card` | Cria ou lê work items a partir de linguagem natural, com critérios de aceite e tags existentes do projeto |

**Variáveis de ambiente:** `AZURE_URL_BOARD`, `AZURE_USER_API_KEY`.

#### Obtendo as credenciais do Azure DevOps

1. Abra o board do Azure DevOps no navegador.
2. Copie a URL do board e separe tudo que estiver à esquerda de `/_boards`, sem a barra (ex.: `https://empresa.visualstudio.com/Projeto`). Use como `AZURE_URL_BOARD`.
3. Abra o menu **"User Settings"** no topo da página e escolha **"Personal access tokens"**.
4. Clique em **"New Token"**, marque **"Read, write, & manage"** em **"Work Items"** e clique em **"Create"**.
5. Copie o token gerado e use como `AZURE_USER_API_KEY`.

---

### `board-plugin` — Trello

| Skill | O que faz |
|---|---|
| `trello-card` | Cria, lê, atualiza, move, arquiva e lista cards; gerencia checklists. Board e coluna resolvidos por nome |
| `generate-card-trello` | Analisa documentos/specs e gera cards com user story, critérios de aceite (Dado/Quando/Então) e especificação técnica |

> **Importante:** a `generate-card-trello` só funciona em conjunto com a `trello-card`, para a qual delega a criação do card. As duas estão no mesmo plugin e usam as mesmas variáveis.

**Variáveis de ambiente:** `TRELLO_API_KEY`, `TRELLO_TOKEN`.

#### Obtendo as credenciais do Trello

1. Faça login no Trello e acesse [https://trello.com/power-ups/admin/](https://trello.com/power-ups/admin/).
2. Clique em **"Novo"**, preencha os campos e crie o aplicativo.
3. Clique em **"Gerar nova chave de API"** e copie a **"Chave de API"** (`TRELLO_API_KEY`).
4. Acesse a URL abaixo (substituindo os colchetes; espaços no nome viram `+`), clique em **"Permitir"** e copie o token exibido (`TRELLO_TOKEN`):

```
https://trello.com/1/authorize?expiration=never&scope=read,write&response_type=token&name=[nome+do+app]&key=[Chave de API]
```

---

## Configurando variáveis de ambiente

**Windows (PowerShell)**

```powershell
$env:NOME_DA_VARIAVEL = "valor"                                                  # só na sessão atual
[Environment]::SetEnvironmentVariable("NOME_DA_VARIAVEL", "valor", "User")      # persistente (usuário atual)
```

**macOS / Linux (bash/zsh)**

```bash
export NOME_DA_VARIAVEL="valor"                                  # só na sessão atual
echo 'export NOME_DA_VARIAVEL="valor"' >> ~/.zshrc && source ~/.zshrc   # persistente
```

Talvez seja necessário reiniciar o Claude Code para que ele enxergue as novas variáveis. Para testar, peça ao Claude Code para executar:<br>
`NOME_DA_VARIAVEL="${NOME_DA_VARIAVEL:?Variável NOME_DA_VARIAVEL não encontrada}"`
