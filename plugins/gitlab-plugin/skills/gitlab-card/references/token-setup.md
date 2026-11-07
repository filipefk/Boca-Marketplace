# Configuração de acesso ao GitLab

A skill fala diretamente com a API REST v4 da instância da empresa. Não usa OAuth nem a CLI `glab` — apenas um Personal Access Token enviado no header `PRIVATE-TOKEN`.

## 1. Gerar o Personal Access Token

Na instância do GitLab da empresa:

1. No canto superior direito, clique no seu avatar → **Edit profile**.
2. No menu lateral, abra **Access** → **Personal access tokens**.
3. No menu suspenso **Generate token**, escolha **Legacy token**.
4. Preencha **Token name** (ex.: `claude-code`), opcionalmente **Token description**, e a **Expiration date** (em branco = 365 dias, o máximo padrão).
5. Marque o escopo:
   - **`api`** — leitura **e** escrita. Necessário para criar, atualizar, mover, fechar e comentar.
   - `read_api` sozinho só permite os fluxos de leitura (`ler`, `listar`, `colunas`, `comentarios`).
6. Clique em **Generate token** e copie o valor — ele só é exibido uma vez.

> **Token fine-grained** — desde a 19.2 (GA, inclusive no Free/CE) o mesmo menu **Generate token** oferece **Fine-grained token**, que restringe o token a projetos e ações específicos e cobre todos os endpoints REST. Serve para esta skill desde que conceda, no projeto, leitura e escrita em issues, notes, labels e boards. As instruções desta skill assumem o token legado com escopo `api`.

O token herda as permissões da sua conta: você só enxerga e altera projetos nos quais já tem acesso (papel Reporter para ler, Developer ou superior para escrever issues).

## 2. Definir as variáveis de ambiente

No PowerShell, de forma persistente para o usuário:

```powershell
[Environment]::SetEnvironmentVariable('GITLAB_URL','https://gitlab.olfar.ind.br','User')
[Environment]::SetEnvironmentVariable('GITLAB_TOKEN','<seu-token>','User')
[Environment]::SetEnvironmentVariable('GITLAB_PROJECT','grupo/projeto','User')
```

Observações:

- `GITLAB_URL` é apenas a origem, **sem barra final** e **sem** `/api/v4` — a skill acrescenta o caminho.
- `GITLAB_PROJECT` é opcional. Aceita o caminho completo (`grupo/subgrupo/projeto`) ou o id numérico do projeto. Quando definido, evita ter que informar `projeto:` em toda invocação; o parâmetro `projeto:` e a URL de uma issue sempre têm precedência sobre ele.
- Depois de definir as variáveis, **reinicie o Claude Code** para que ele as enxergue.

## 3. Validar a configuração

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "${GITLAB_URL%/}/api/v4/user"
```

- `200` — token válido.
- `401` — token inválido, revogado ou expirado.
- Erro de conexão/TLS — confira a `GITLAB_URL` e o acesso à rede interna (VPN).

## Segurança

- Nunca cole o token na conversa, em um arquivo do projeto ou em uma URL.
- O GitLab aceita o token apenas em header; a skill nunca o coloca em query string.
- Se o token vazar, revogue-o na mesma tela (**Access** → **Personal access tokens**) e gere outro.
