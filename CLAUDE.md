# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Este é um **marketplace de plugins do Claude Code** (`boca-marketplace`). Não é uma aplicação de software tradicional — não há build, lint, testes ou processo de compilação. O conteúdo do repositório é inteiramente declarativo: arquivos `.json` de manifesto e arquivos `SKILL.md` em Markdown que instruem o Claude Code a executar tarefas específicas (chamadas de API via `curl`, interpretação de linguagem natural, confirmação com o usuário).

Não execute build/lint/test neste repositório — eles não existem e não fazem sentido aqui.

## Estrutura

```
.claude-plugin/marketplace.json          # manifesto do marketplace (lista os plugins e onde encontrá-los)
plugins/
  google-plugin/
    .claude-plugin/plugin.json           # manifesto do plugin
    skills/
      google-drive/                      # listar pastas do Drive; references/oauth-setup.md é o setup OAuth compartilhado por todas as google-*
      google-docs/                       # ler, criar e editar documentos (references/batchupdate-requests.md)
      google-sheets/                     # ler e editar planilhas (references/batchupdate-receitas.md)
      google-slides/                     # criar e editar apresentações (references/batchupdate-requests.md)
  gitlab-plugin/
    .claude-plugin/plugin.json
    skills/
      gitlab-card/                       # issues no board (references: api-reference, card-rules, card-models, token-setup)
      gitlab-pr/                         # merge requests e revisão (references: api-reference, mr-models, review-rules, token-setup)
      gitlab-release/                    # releases e notas de release (references: api-reference, release-notes, token-setup)
      gitlab-wiki/                       # páginas da Wiki (references: api-reference, wiki-rules, licoes-aprendidas, token-setup)
  azure-plugin/
    .claude-plugin/plugin.json
    skills/
      azure-card/                        # work items no Azure DevOps (references: card-rules, card-models em HTML)
  board-plugin/
    .claude-plugin/plugin.json
    skills/
      trello-card/                       # cards no Trello (references: card-rules, card-models em Markdown)
      generate-card-trello/              # fluxo multi-fase (Analyze → Discuss → Structure → Create); delega a criação para a trello-card
```

Cada skill fica em `plugins/<plugin>/skills/<skill>/SKILL.md`, com arquivos de apoio em `references/`.

Ao adicionar um novo plugin, ele precisa ser registrado em `.claude-plugin/marketplace.json` (campo `plugins`, com `name`, `description` e `source` apontando para a pasta do plugin) e ter seu próprio `.claude-plugin/plugin.json`.

## Padrões importantes a preservar ao editar ou criar skills

- **Frontmatter** com `name` e `description` — a `description` é o que o Claude Code usa para decidir quando invocar a skill, então deve ser específica sobre o que a skill faz e quais variáveis de ambiente requer.
- **Sempre confirmar antes de ações que escrevem/alteram dados** (criar, atualizar, mover, arquivar, apagar, publicar) usando `AskUserQuestion` com opções claras. Nunca chamar a API de escrita sem essa confirmação.
- **Resolução por nome, não por ID**: boards, colunas, projetos e páginas são resolvidos dinamicamente via API a partir do nome informado pelo usuário, nunca hardcoded.
- **Credenciais vêm de variáveis de ambiente**, nunca de arquivos do repositório. Os scripts bash usam a forma `VAR="${VAR:?Variável VAR não encontrada}"` para falhar cedo.
- **Payloads JSON via arquivo temporário** (`mktemp` + `--data-binary "@arquivo"`), não inline no `-d`, para evitar problemas de escaping.
- Erros de API devem ser exibidos ao usuário, não engolidos.
- **Dependências entre skills ficam no mesmo plugin**: as skills `google-*` referenciam `../google-drive/references/oauth-setup.md`; as `gitlab-*` compartilham as mesmas credenciais; a `generate-card-trello` delega a criação do card para a `trello-card`. Ao mover uma skill de plugin, verifique esses caminhos relativos.

## Variáveis de ambiente necessárias

| Plugin | Skills | Variáveis |
|---|---|---|
| `google-plugin` | `google-drive`, `google-docs`, `google-sheets`, `google-slides` | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN` |
| `gitlab-plugin` | `gitlab-card`, `gitlab-pr`, `gitlab-release`, `gitlab-wiki` | `GITLAB_URL`, `GITLAB_TOKEN` (opcional `GITLAB_PROJECT`) |
| `azure-plugin` | `azure-card` | `AZURE_URL_BOARD`, `AZURE_USER_API_KEY` |
| `board-plugin` | `trello-card`, `generate-card-trello` | `TRELLO_API_KEY`, `TRELLO_TOKEN` |

Detalhes de como obter cada credencial estão no [README.md](README.md).
