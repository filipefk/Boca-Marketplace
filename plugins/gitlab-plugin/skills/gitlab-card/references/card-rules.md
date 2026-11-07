# Regras de criação de cards no GitLab

## Coluna do card

Cada coluna do board corresponde a uma label. O significado de cada uma:

| Coluna    | Significado                                  |
|-----------|-----------------------------------------------|
| `Backlog` | Card aguardando ser priorizado                |
| `To Do`   | Card priorizado aguardando ser atendido       |
| `Doing`   | Card sendo implementado                       |
| `Done`    | PR gerado aguardando revisão                  |

A resolução da coluna (por parâmetro `coluna:`, padrão `Backlog` na criação, listagem, movimentação etc.) segue as regras da parte **D** do Passo comum em [SKILL.md](../SKILL.md).

## Tipo do card

Infira o tipo a partir da descrição e aplique a label de mesmo nome no projeto:

| Tipo             | Quando usar                                              | Cor sugerida |
|------------------|-----------------------------------------------------------|--------------|
| `Bug`            | Problema, falha, erro, comportamento incorreto            | `#D9534F`    |
| `Conf. Ambiente` | Configuração de ambiente (infra, variáveis, deploy, setup) | `#F0AD4E`    |
| `Documentação`   | Criação de documentação                                    | `#428BCA`    |
| `Melhoria`       | Melhoria, ajuste, refatoração                              | `#5CB85C`    |
| `Sanitizar`      | Limpeza/sanitização de dados                               | `#8E44AD`    |
| `Segurança`      | Vulnerabilidade ou questão de segurança                    | `#B00020`    |
| `Tarefa`         | Demais casos: tarefa técnica                               | `#FFD800`    |

A label só é aplicada se **já existir** no projeto. Se não existir, crie a issue sem ela e avise no relatório final — o GitLab criaria a label silenciosamente, poluindo o projeto. As cores acima servem para quem quiser criá-las manualmente em **Project → Manage → Labels**.

A label de tipo é independente da label de coluna: uma issue costuma ter as duas (ex.: `Bug` + `Doing`).

## Título (`title`)

Frase curta e objetiva (máximo 80 caracteres) resumindo a essência da descrição, seguida obrigatoriamente do sufixo ` (Criado pelo Claude Code)`.

Exemplo: `Corrigir login inválido (Criado pelo Claude Code)`

Não coloque o tipo no título — ele já é expresso pela label.

## Descrição (`description`)

Use os modelos em [card-models.md](card-models.md). O GitLab renderiza Markdown no campo `description`.

## Boas práticas de conteúdo

Ao redigir o título e a descrição do card, siga estes princípios:

- **Título claro**: deixe evidente o que será feito, não apenas o tema (ex.: "Corrigir timeout no upload de anexos", em vez de "Upload de anexos").
- **Descrição com o porquê**: explique o problema ou a necessidade por trás do card, não só o quê fazer — o porquê ajuda quem for implementar a decidir entre alternativas.
- **Critério de aceite**: a descrição deve deixar claro como saber que o card está pronto. Inclua a seção "Critérios de Aceite" (veja os modelos em [card-models.md](card-models.md)) mesmo em Bugs e Tasks, quando fizer sentido.
- **Tamanho**: um card deve caber em poucos dias de trabalho. Card grande gera branch longo, que acumula conflito de merge. Se a descrição sugerir um escopo maior que isso, proponha quebrar em cards menores usando `AskUserQuestion` antes de criar — liste a divisão sugerida e deixe o usuário confirmar ou ajustar.

## Confirmação antes de escrever

Sempre use `AskUserQuestion` antes de qualquer chamada que altere dados (criar, atualizar, mover, fechar, reabrir, comentar). Se o usuário cancelar, encerre sem chamar a API.

## Exibição do card para confirmação

```
Projeto: <path_with_namespace>
Board:   <nome do board>
Coluna:  <nome da coluna>
Tipo:    <tipo inferido>
Título:  <título>
Prazo:   <due_date, se informado>

Descrição:
<conteúdo em texto legível>
```
