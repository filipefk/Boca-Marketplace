# Modelos de cards (HTML)

O Azure DevOps renderiza HTML nos campos de texto. Use os modelos abaixo como base.

> **Todos os modelos possuem a seção "Critérios de Aceite", que é obrigatória.** Ela nunca pode ser omitida nem ficar vazia — se a descrição do usuário não trouxer critérios explícitos, infira critérios objetivos e verificáveis a partir do contexto.

## User Story — campo `System.Description`

```html
<p><b>Como</b> [ator inferido], <b>quero</b> [ação principal] <b>para</b> [benefício].</p>
<h3>Critérios de Aceite</h3>
<ul>
  <li>[critério 1 extraído ou inferido da descrição]</li>
  <li>[critério 2]</li>
</ul>
```

## Bug — campo `Microsoft.VSTS.TCM.ReproSteps`

> Cards de Bug **não têm** o campo "Description". Use `Microsoft.VSTS.TCM.ReproSteps` para toda a descrição, inclusive os critérios de aceite.

```html
<h3>Comportamento Atual</h3>
<p>[o que está acontecendo de errado, conforme descrito]</p>
<h3>Comportamento Esperado</h3>
<p>[o que deveria acontecer]</p>
<h3>Critérios de Aceite</h3>
<ul>
  <li>[critério que comprova a correção do bug]</li>
  <li>[critério de não regressão, se aplicável]</li>
</ul>
```

## Task — campo `System.Description`

```html
<h3>Objetivo</h3>
<p>[o que deve ser feito]</p>
<h3>Detalhes</h3>
<p>[informações relevantes da descrição]</p>
<h3>Critérios de Aceite</h3>
<ul>
  <li>[critério 1 que define a task como concluída]</li>
  <li>[critério 2]</li>
</ul>
```
