# Configuração de acesso ao GitLab

Esta skill usa **as mesmas credenciais das skills `gitlab-card`, `gitlab-pr` e `gitlab-wiki`**: um Personal Access Token enviado no header `PRIVATE-TOKEN` para a API REST v4 da instância da empresa. Não usa OAuth nem a CLI `glab`.

Se a `gitlab-card` já está funcionando, não há nada a configurar — pule para a seção "Validar".

## 1. Gerar o Personal Access Token

Na instância do GitLab da empresa:

1. No canto superior direito, clique no seu avatar → **Edit profile**.
2. No menu lateral, abra **Access** → **Personal access tokens**.
3. No menu suspenso **Generate token**, escolha **Legacy token**.
4. Preencha **Token name** (ex.: `claude-code`), opcionalmente **Token description**, e a **Expiration date** (em branco = 365 dias, o máximo padrão).
5. Marque o escopo:
   - **`api`** — leitura **e** escrita. Necessário para criar, atualizar e apagar releases e links.
   - `read_api` sozinho só permite os fluxos de leitura (`listar`, `ler`, `mudancas`, `links`).
6. Clique em **Generate token** e copie o valor — ele só é exibido uma vez.

> **Token fine-grained** — desde a 19.2 (GA, inclusive no Free/CE) o mesmo menu **Generate token** oferece **Fine-grained token**, que restringe o token a projetos e ações específicos. Serve para esta skill desde que conceda, no projeto, leitura e escrita em releases e leitura do repositório (tags, compare, commits), de merge requests, de issues e de milestones. As instruções desta skill assumem o token legado com escopo `api`.

O token herda as permissões da sua conta no projeto:

| Ação | Papel mínimo no projeto |
|---|---|
| Ler releases, tags, compare, MRs e issues | Reporter (ou Guest em projeto público/interno) |
| Criar, atualizar e apagar release; adicionar/remover links | Developer |
| Criar release em **tag protegida** | Permissão de criar aquela tag protegida (em geral Maintainer) |

## 2. Definir as variáveis de ambiente

No PowerShell, de forma persistente para o usuário:

```powershell
[Environment]::SetEnvironmentVariable('GITLAB_URL','https://gitlab.olfar.ind.br','User')
[Environment]::SetEnvironmentVariable('GITLAB_TOKEN','<seu-token>','User')
[Environment]::SetEnvironmentVariable('GITLAB_PROJECT','grupo/projeto','User')
```

Observações:

- `GITLAB_URL` é apenas a origem, **sem barra final** e **sem** `/api/v4` — a skill acrescenta o caminho.
- `GITLAB_PROJECT` é opcional. Aceita o caminho completo (`grupo/subgrupo/projeto`) ou o id numérico. O parâmetro `projeto:` e a URL de uma release sempre têm precedência sobre ele.
- Depois de definir as variáveis, **reinicie o Claude Code** para que ele as enxergue.

## 3. Validar a configuração

Token válido:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" "${GITLAB_URL%/}/api/v4/user"
```

- `200` — token válido.
- `401` — token inválido, revogado ou expirado.
- Erro de conexão/TLS — confira a `GITLAB_URL` e o acesso à rede interna (VPN).

Acesso às releases do projeto:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" "${GITLAB_URL%/}/api/v4/projects/${PROJECT_ID}/releases?per_page=1"
```

- `200` — ok (mesmo que a lista venha vazia).
- `404` — projeto inexistente, sem acesso, ou releases desabilitadas no projeto.
- `403` — token sem escopo suficiente.

## Segurança

- Nunca cole o token na conversa, em um arquivo do projeto ou em uma URL.
- O GitLab aceita o token em header; a skill nunca o coloca em query string.
- Se o token vazar, revogue-o na mesma tela (**Access** → **Personal access tokens**) e gere outro.
- Lembre que **criar uma release é publicar**: ela fica visível para todos com acesso ao projeto e pode notificar quem acompanha releases. Por isso toda escrita pede confirmação.
