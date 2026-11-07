# Modelos de descrição de issue

O GitLab renderiza Markdown no campo `description`. Use os modelos abaixo como base.

## Story

```markdown
**Como** [ator inferido], **quero** [ação principal] **para** [benefício].

## Critérios de Aceite
- [critério 1]
- [critério 2]
```

## Bug

```markdown
## Comportamento Atual
[o que acontece hoje]

## Comportamento Esperado
[o que deveria acontecer]

## Passos para Reproduzir
1. [passo 1]
2. [passo 2]
```

## Task

```markdown
## Objetivo
[o que precisa ser feito]

## Detalhes
[contexto técnico, restrições, pontos de atenção]
```

## Checklist

O GitLab CE não tem objeto de checklist como o Trello. O equivalente é a **task list** nativa do Markdown, acrescentada ao final da descrição:

```markdown
## Tarefas
- [ ] Primeiro item
- [ ] Segundo item
- [x] Item já concluído
```

Regras:

- Use o valor do parâmetro `checklist:` como título da seção (`## Tarefas`, `## Critérios`, etc.).
- Os itens começam sempre desmarcados (`- [ ]`), exceto quando o usuário indicar que algo já está pronto.
- O GitLab exibe o progresso automaticamente ao lado do título da issue (ex.: `1/3`) e permite marcar os itens direto na interface, sem editar a descrição.
- Ao atualizar a descrição de uma issue existente, preserve a seção de checklist se ela não for o alvo da alteração — sobrescrever a descrição apaga o progresso já marcado.
