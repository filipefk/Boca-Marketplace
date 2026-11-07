# Formato XML do draw.io (mxGraph)

## Estrutura geral

Um arquivo `.drawio` é sempre um XML `mxfile`, com uma ou mais páginas (`diagram`):

```xml
<mxfile host="Electron" version="24.0.0">
  <diagram id="AbCdEfGh12345" name="Page-1">
    <mxGraphModel dx="800" dy="600" grid="1" gridSize="10" guides="1" tooltips="1"
        connect="1" arrows="1" fold="1" page="1" pageScale="1"
        pageWidth="850" pageHeight="1100" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
        <!-- formas e conexões entram aqui, todas com parent="1" (ou dentro de um grupo) -->
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

- `mxfile` pode conter **múltiplos** elementos `diagram` (múltiplas páginas do mesmo arquivo). Se o pedido do usuário citar uma página pelo nome, use o `diagram` com esse `name`; senão, use o primeiro.
- Dentro de `root`, as células `id="0"` (raiz) e `id="1"` (camada default, `parent="0"`) são obrigatórias e nunca devem ser removidas ou reaproveitadas para outra coisa.
- Toda forma/conexão nova entra como um novo `mxCell` filho de `root`, normalmente com `parent="1"`.

## Vértices (formas)

```xml
<mxCell id="node1" value="Minha caixa" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;"
    vertex="1" parent="1">
  <mxGeometry x="120" y="80" width="160" height="60" as="geometry" />
</mxCell>
```

- `value` — o texto/label exibido (pode conter HTML simples se `html=1` estiver no style).
- `vertex="1"` marca a célula como forma (retângulo, losango, círculo etc.).
- `mxGeometry` define posição (`x`, `y`) e tamanho (`width`, `height`) em pixels, relativo à página. `as="geometry"` é fixo.

## Arestas (conexões)

```xml
<mxCell id="edge1" value="" style="edgeStyle=orthogonalEdgeStyle;rounded=0;html=1;"
    edge="1" parent="1" source="node1" target="node2">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

- `edge="1"` marca a célula como conexão.
- `source`/`target` referenciam o `id` das células conectadas (ambos devem existir).
- `mxGeometry relative="1"` é o padrão para arestas (a posição é derivada de `source`/`target`, não de `x`/`y`).

## Regras de `id`

- Todo `id` novo criado pela skill deve ser único no diagrama (não reaproveitar `id` já existente lido na inspeção).
- Não há formato obrigatório — qualquer string sem espaços serve (ex: `node1`, `edge_a_b`, um UUID curto). Evite `0` e `1`, reservados.

## Mini-catálogo de `style` (tokens mais comuns)

| Token | Efeito |
|---|---|
| `rounded=1` / `rounded=0` | cantos arredondados ou retos |
| `whiteSpace=wrap` | quebra de linha automática no texto |
| `html=1` | permite HTML simples no `value` |
| `fillColor=#RRGGBB` | cor de preenchimento |
| `strokeColor=#RRGGBB` | cor da borda/linha |
| `fontSize=12` | tamanho da fonte |
| `fontColor=#RRGGBB` | cor do texto |
| `ellipse` | forma oval (usar como prefixo do style, ex: `ellipse;whiteSpace=wrap;html=1;`) |
| `rhombus` | losango |
| `edgeStyle=orthogonalEdgeStyle` | conexão em ângulo reto |
| `edgeStyle=none` | conexão em linha reta |
| `startArrow=none` / `endArrow=none` | remove seta de uma ponta |
| `dashed=1` | linha tracejada |

Não é uma lista exaustiva — o draw.io aceita qualquer combinação de `chave=valor;` separados por `;`. Ao editar uma célula existente, prefira herdar o `style` atual e só alterar o token necessário, em vez de reescrever do zero.

## Compressão do conteúdo do `<diagram>`

Por padrão, quando o draw.io salva um arquivo, o conteúdo de cada `<diagram>` **não fica como XML legível** — vem comprimido e em base64:

```xml
<diagram id="AbCdEfGh12345" name="Page-1">7Vpbc6M2FP41...</diagram>
```

Esquema de compressão (nessa ordem, para gerar o texto acima a partir do XML do `mxGraphModel`):

1. `encodeURIComponent(xml)` — percent-encoding.
2. **raw deflate** (RFC 1951, sem cabeçalho zlib/gzip) sobre os bytes UTF-8 do resultado do passo 1.
3. base64 do resultado comprimido.

Se o atributo `compressed="false"` estiver presente no `<diagram>`, o conteúdo já é o XML puro (sem os 3 passos acima) — trate como texto direto.

### Descomprimir (leitura) — PowerShell

O ambiente (Git Bash) não tem `zlib`/`python` disponível para raw deflate. Use a tool **PowerShell**, que tem acesso a `System.IO.Compression.DeflateStream` (implementa exatamente RFC 1951):

```powershell
$b64 = Get-Content -Raw "<arquivo com o conteúdo base64 do <diagram>>"
$compressed = [Convert]::FromBase64String($b64.Trim())
$ms = New-Object System.IO.MemoryStream(,$compressed)
$ds = New-Object System.IO.Compression.DeflateStream($ms, [System.IO.Compression.CompressionMode]::Decompress)
$sr = New-Object System.IO.StreamReader($ds, [System.Text.Encoding]::UTF8)
$encoded = $sr.ReadToEnd()
$xml = [System.Uri]::UnescapeDataString($encoded)
Set-Content -Path "<arquivo de saída>" -Value $xml -NoNewline
```

### Comprimir (escrita) — não necessário nesta skill

Ver decisão abaixo: esta skill **nunca grava conteúdo comprimido**. O snippet de compressão (deflate) não precisa ser implementado — sempre grave com `compressed="false"` e XML puro dentro do `<diagram>`.

## Decisão de design: sempre gravar sem compressão

Ao editar ou criar um arquivo `.drawio`, a skill sempre escreve o(s) `<diagram>` com o atributo `compressed="false"` e o XML do `mxGraphModel` em texto puro, **independentemente de como o arquivo original estava** (mesmo que a leitura tenha exigido descomprimir).

Motivo: reimplementar o deflate de escrita corretamente (percent-encoding + raw deflate byte-a-byte compatível com o que o draw.io espera) tem risco real de gerar um arquivo corrompido, sem forma fácil de validar antes de o usuário abrir no app. Gravar sem compressão é 100% seguro: o draw.io abre e edita normalmente diagramas não comprimidos, e volta a comprimir sozinho na próxima vez que o usuário salvar pelo app — sem qualquer perda de conteúdo.
