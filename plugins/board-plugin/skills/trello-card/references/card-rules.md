# Regras de criação de cards no Trello

## Tipo do card

Inferir com base na descrição para definir o **label** do card:

| Tipo    | Quando usar                                               | Cor do label |
|---------|-----------------------------------------------------------|--------------|
| `Bug`   | Problema, falha, erro ou comportamento incorreto          | `red`        |
| `Story` | Funcionalidade do ponto de vista do usuário ou negócio    | `green`      |
| `Task`  | Demais casos: tarefa técnica, melhoria, ajuste, etc.      | `blue`       |

## Título (name)

Frase curta e objetiva (máximo 80 caracteres) resumindo a essência da descrição, seguida obrigatoriamente do sufixo ` (Criado pelo Claude Code)`.

Exemplo: `Corrigir login inválido (Criado pelo Claude Code)`.

## Descrição

O Trello usa **Markdown** no campo `desc`. Use os modelos em [card-models.md](card-models.md).

## Confirmação antes de criar

Sempre exibir o card montado para confirmação usando `AskUserQuestion` com as opções **Criar card** e **Cancelar** antes de chamar a API. Se cancelar, encerrar sem fazer nada.

## Exibição do card para confirmação

```
Tipo:    <tipo inferido>
Título:  <título>

Descrição:
<conteúdo em texto legível>
```
