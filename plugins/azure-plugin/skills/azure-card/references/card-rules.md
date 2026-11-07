# Regras de criação de cards

## Tipo do work item

Inferir com base na descrição:

| Tipo | Quando usar |
|------|-------------|
| `Bug` | Problema, falha, erro ou comportamento incorreto |
| `User Story` | Funcionalidade do ponto de vista do usuário ou negócio |
| `Task` | Demais casos: tarefa técnica, melhoria, ajuste, etc. |

## Título

Frase curta e objetiva (máximo 80 caracteres) resumindo a essência da descrição, seguida obrigatoriamente do sufixo ` (Criado pela Claudinha)`.

Exemplo: `Corrigir login inválido (Criado pela Claudinha)`.

## Campo de descrição por tipo

- **User Story** e **Task**: usar o campo `System.Description`
- **Bug**: usar o campo `Microsoft.VSTS.TCM.ReproSteps` (campo "Repro Steps")
  - Cards de Bug **não possuem** o campo "Description" no Azure DevOps
  - Todo o conteúdo descritivo do bug deve ir em `Microsoft.VSTS.TCM.ReproSteps`

## Critérios de aceite (obrigatório)

Todo card, de qualquer tipo, **deve** conter a seção "Critérios de Aceite" na descrição, conforme os modelos em [card-models.md](card-models.md).

- Se o usuário informar critérios, use-os.
- Se não informar, infira critérios objetivos e verificáveis a partir da descrição (no mínimo um).
- Nunca criar um card sem critérios de aceite.

## Tags

Usar preferencialmente as tags **já existentes** no projeto (campo `System.Tags`), escolhidas por afinidade com o conteúdo do card.

- Buscar as tags existentes via API antes de montar o card (ver "Buscar tags existentes" no [SKILL.md](../SKILL.md)).
- Selecionar as tags cujo nome tenha relação clara com o card (tipo, módulo, área, tecnologia, cliente, etc.). Comparação case-insensitive; usar o nome exatamente como cadastrado.
- Não inventar tags novas. Se nenhuma tag existente tiver afinidade, criar o card sem tags.
- Se o usuário pedir explicitamente uma tag que não existe, usá-la mesmo assim, avisando na confirmação que ela será criada.
- Múltiplas tags são enviadas em um único valor separado por `; ` (ex.: `Backend; Login`).

## Relacionamento entre cards

**Não criar cards filhos** (relação pai/filho — `System.LinkTypes.Hierarchy-*`). Quando for necessário relacionar o card a um ou mais cards, usar **sempre** o relacionamento **Related** (`System.LinkTypes.Related`).

- Se o usuário pedir um card "filho de" outro, criar o card com relacionamento Related ao card informado e avisar na confirmação que cards filhos não são suportados.
- Se o usuário pedir vários cards juntos (ex.: "uma User Story e duas Tasks"), criar todos de forma independente e relacioná-los via Related ao card principal.

## Formato HTML

O Azure DevOps renderiza HTML nos campos de texto. Use HTML simples conforme os modelos em [card-models.md](card-models.md).

## Confirmação antes de criar

Sempre exibir o card montado para confirmação usando `AskUserQuestion` com as opções **Criar card** e **Cancelar** antes de chamar a API. Se cancelar, encerrar sem fazer nada.

## Exibição do card para confirmação

```
Tipo:         <tipo>
Título:       <título>
Tags:         <tags separadas por "; " ou "(nenhuma)">
Relacionado:  <IDs dos cards relacionados ou "(nenhum)">

Descrição:
<conteúdo em texto legível, sem tags HTML, incluindo os critérios de aceite>
```
