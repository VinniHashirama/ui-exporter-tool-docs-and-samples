# O contrato: UIIR

Spec técnica da fronteira entre o plugin do Figma e o importador da Unity. Público: devs.
Para as regras que o designer segue, ver [`figma-conventions.md`](figma-conventions.md).

O schema formal vive no repositório do plugin: [`schema/uiir.schema.json`](https://github.com/VinniHashirama/ui-exporter-tool-figma-plugin/blob/main/schema/uiir.schema.json) — em caso de
divergência, **o schema manda**. Este doc explica o *porquê* de cada decisão.

---

## O pacote `.uiexport`

Um zip com nome `<NomeDaTela>.uiexport`:

```
HomeMenu.uiexport
├─ ui.json              # o UIIR, valida contra schema/uiir.schema.json
└─ images/
   ├─ hero@2x.png
   └─ logo@2x.png
```

Sem outras entradas. O importador **rejeita** o pacote inteiro se encontrar qualquer coisa
fora desse formato — ver [Segurança](#segurança).

---

## Princípios do IR

**1. Neutro nas duas pontas.** Nenhum campo é nomeado por Figma ou por Unity. Não existe
`figmaNodeId` nem `rectTransform` — existe `id` e `rect`. Isso é o que permite trocar o
backend (UI Toolkit) ou a origem (Sketch, Penpot) sem reescrever o outro lado.

**2. Valores resolvidos, tokens informativos.** Cada node carrega o valor final (`color:
"#FF6A00"`) *e*, quando aplicável, o token de origem (`token: "primary"`). O importador usa
o valor; o token serve para diagnóstico e para theming futuro. O importador nunca precisa
resolver uma tabela de tokens para funcionar.

**3. Sem estado implícito.** `layoutChild` só existe quando o pai tem `layout`.
`constraints` só é aplicado quando o pai **não** tem `layout`. Nada é inferido de contexto
distante.

**4. Ordem CSS em quadrupletas.** `padding` e `nineSlice` são arestas —
`[top, right, bottom, left]`. `cornerRadius` são cantos —
`[topLeft, topRight, bottomRight, bottomLeft]`. Ambas seguem a ordem horária do CSS.

**5. Y para baixo.** `rect` usa origem no canto superior-esquerdo com Y crescendo para
baixo, porque é a convenção de toda ferramenta de design. A inversão para o espaço da Unity
acontece num único lugar: `RectSolver`.

---

## Identidade e reconciliação

`node.id` é o campo mais importante do schema.

O importador mantém dois arquivos por tela:

```
Assets/UI/Generated/<Nome>_Base.prefab   ← da ferramenta, sobrescrito a cada import
Assets/UI/Screens/<Nome>.prefab          ← Prefab Variant, do dev, nunca tocado
```

Um Prefab Variant rastreia seus overrides pelo **`fileID` local de cada objeto no prefab
base**. Se o import destruísse e recriasse os GameObjects do `_Base`, todos os `fileID`
mudariam e os overrides do Variant virariam órfãos — o dev perderia scripts, referências e
ajustes, silenciosamente e sem erro no console.

Por isso o builder **nunca recria**. Ele reconcilia:

1. `PrefabUtility.LoadPrefabContents` do `_Base`, se existir.
2. Indexa os GameObjects existentes por `FigmaNodeRef.nodeId`.
3. Caminha o IR: node com match → **reusa** o GameObject (reparent/reorder); sem match → cria.
4. GameObject cujo `nodeId` desapareceu do IR → remove, e **lista no report**.
5. `PrefabUtility.SaveAsPrefabAsset` + `UnloadPrefabContents`.

`fileID` preservado ⇒ override do dev sobrevive.

A consequência prática para o designer: **renomear uma layer é seguro, deletar e recriar não
é**. Uma layer recriada tem `id` novo, então para a ferramenta é um objeto diferente — o
objeto antigo é removido e o trabalho que o dev tinha pendurado nele vai com ele. O report
sempre lista as remoções, e o `DiffPreview` pede confirmação antes de gravar qualquer coisa.

---

## Mapeamentos

### Auto Layout → UGUI

| IR | UGUI |
|---|---|
| `layout.mode: HORIZONTAL` / `VERTICAL` | `HorizontalLayoutGroup` / `VerticalLayoutGroup` |
| `layout.mode: WRAP` | `GridLayoutGroup` — aproximação, reportada |
| `layout.spacing` | `.spacing` |
| `layout.padding` | `.padding` (converter ordem CSS → `RectOffset`) |
| `layout.primaryAlign` MIN/CENTER/MAX | `.childAlignment`, eixo primário |
| `layout.primaryAlign: SPACE_BETWEEN` | espaçadores gerados + aviso |
| `layout.counterAlign` | `.childAlignment`, eixo cruzado (`BASELINE` → `MIN`) |
| `layout.sizing.*: HUG` | `ContentSizeFitter` = `PreferredSize` no eixo |
| `layout.sizing.*: FIXED` | `ContentSizeFitter` = `Unconstrained` + `sizeDelta` |
| `layoutChild.sizing.*: FILL` | `LayoutElement.flexible{Width,Height} = 1` + `childForceExpand` |
| `layoutChild.sizing.*: FIXED` | `LayoutElement.preferred{Width,Height}` + `childControlSize = true` |

Filho de container com `layout` **não** passa pelo `RectSolver` — o LayoutGroup dirige o
`RectTransform`. Aplicar posição na mão ali gera briga entre os dois e layout instável.

### Constraints → anchors

Só para filhos de container **sem** `layout`. O eixo vertical inverte: `MIN` no IR é o topo,
que na Unity é `anchorMax.y`.

| IR | `anchorMin` → `anchorMax` | Posicionamento |
|---|---|---|
| `MIN` | `0` → `0` | `anchoredPosition` a partir da borda de origem |
| `MAX` | `1` → `1` | `anchoredPosition` negativo, a partir da borda oposta |
| `CENTER` | `0.5` → `0.5` | `anchoredPosition` = delta do centro |
| `STRETCH` | `0` → `1` | `offsetMin` / `offsetMax` |
| `SCALE` | `x/W` → `(x+w)/W` | `offsetMin` = `offsetMax` = `0` |

### Texto → TextMeshPro

| IR | TMP |
|---|---|
| `font.family` + `font.style` | `TMP_FontAsset` via `FontMap`; faltando → fallback + aviso |
| `size` | `fontSize` |
| `color` | `color` |
| `alignHorizontal` × `alignVertical` | `TextAlignmentOptions` (produto das duas) |
| `lineHeight.unit: PIXELS` | `lineSpacing = (desejado − defaultDaFonte) / size × 100` |
| `letterSpacing` | `characterSpacing` |
| `autoResize: NONE` + `truncation: ELLIPSIS` | `overflowMode = Ellipsis` |
| `autoResize: WIDTH_AND_HEIGHT` | `ContentSizeFitter` = `PreferredSize` |
| `maxLines` | `maxVisibleLines` |
| `case` | `fontStyle` (`UpperCase` / `LowerCase` / `SmallCaps`) |

**Aviso honesto sobre texto:** `lineSpacing` e `characterSpacing` do TMP não estão nas mesmas
unidades do Figma, e a conversão depende das métricas da fonte concreta. As fórmulas acima
são o ponto de partida, **não** a resposta final — precisam de calibração empírica contra
uma cena de teste, por família de fonte. Além disso, a caixa de texto do Figma e a baseline
do TMP não coincidem exatamente: espere ±1–2px de diferença vertical. Não vale perseguir
pixel-perfect em texto; vale garantir que a fonte, o tamanho, a cor e o alinhamento estão
certos e que o texto não vaza da caixa.

### Componentes → prefabs do kit

`component.canonicalName` resolve para o prefab cujo `UIKitComponent.canonicalName` casa.
A descoberta é por **varredura do projeto**, não por configuração manual — a `UIMappingTable`
guarda apenas exceções e overrides por jogo.

A instância é criada com `PrefabUtility.InstantiatePrefab` **aninhada** dentro do `_Base`.
Label, ícone e rect entram como *property overrides* da instância, então melhoria no prefab
do kit propaga para todas as telas de graça.

O prefab do kit declara seus próprios pontos de injeção via `UIKitComponent.slots` — o
builder nunca adivinha caminho de filho. Ver [`prefab-kit.md`](prefab-kit.md).

---

## Versionamento

`schemaVersion` é semver e o importador aplica a regra:

| Situação | Comportamento |
|---|---|
| MAJOR igual, MINOR ≤ do importador | Importa |
| MAJOR igual, MINOR > do importador | Importa, avisa que o pacote é mais novo |
| MAJOR diferente | **Recusa** com mensagem clara |

- **PATCH** — correção de descrição, sem efeito em dado.
- **MINOR** — campo opcional novo, ou valor novo em enum tolerado por default.
- **MAJOR** — campo removido/renomeado, obrigatoriedade nova, mudança de semântica.

Recusar é deliberado: adivinhar a intenção de um pacote de outra major gera prefab
silenciosamente errado, que é muito pior que um erro de import.

---

## Fora do escopo do MVP

Achatar em PNG via `#img` (o designer decide):

- gradientes, sombras, blur, blend modes
- vetores complexos e operações booleanas

Não implementado:

- detecção automática de 9-slice — o campo `asset.nineSlice` existe no schema mas só é
  preenchido por anotação explícita
- geração de `SpriteAtlas`
- estados/variantes automáticos — os estados vêm do prefab do kit
- animações e transições
- sistema de localização — `text.locKey` é só registrado
- codegen de view tipada — no MVP o binding é `UIViewRefs.Get<T>(key)`
- backend UI Toolkit
- múltiplos breakpoints por tela
- Figma REST API

---

## Segurança

**Nenhum segredo, por construção.** A rota plugin+zip não usa Personal Access Token do
Figma. O `manifest.json` do plugin declara `networkAccess: { allowedDomains: ["none"] }` —
o plugin não faz nenhuma chamada de rede, e isso é verificável no manifest.

Se a REST API entrar numa versão futura, o token vai em `EditorPrefs` ou variável de
ambiente — **nunca** em asset, repositório ou log — com redação obrigatória em qualquer
report ou mensagem de erro.

**Sem PII.** `source` carrega `fileKey`, `fileName`, timestamp e versão do plugin. Não
carrega nome nem e-mail de quem exportou, deliberadamente.

**O pacote é dado não confiável.** O `ZipReader` valida antes de escrever qualquer byte:

- normaliza cada entry path e rejeita `..`, path absoluto e letra de drive (zip-slip)
- aceita apenas `ui.json` e `images/*.png`
- limita número de entries e tamanho total descomprimido (zip-bomb)
- extrai em pasta temporária e só move para `Assets/` depois de tudo validar

Nomes de layer e conteúdo de texto viram nome de GameObject e conteúdo de TMP: são
sanitizados de caracteres inválidos de path e **nunca** interpretados como caminho de asset.

**Dependências** vêm só de registries oficiais, com versão exata fixada e lockfile
commitado. `npm ci --ignore-scripts` no plugin. Nenhum script de CDN na UI do plugin — a CSP
do Figma bloquearia de todo jeito, mas a regra é explícita. Dependência transitiva nova
passa por revisão antes de entrar no lockfile.

---

## Milestones

| # | Entrega | Status |
|---|---|---|
| **M0** | Schema + docs do contrato | ✅ código pronto; falta a revisão com o dono da Library |
| **M1** | Plugin Figma: traversal, mappers, lint, assets, zip | ✅ 69 testes, saída validada contra o schema |
| **M2** | Kit placeholder na Unity: 15 prefabs + `UIKitComponent`/slots | ✅ gerado por código, não commitado |
| **M3** | Importador: unzip, gate de schema, resolver, solver, builder | ✅ 55 testes EditMode |
| **M4** | Diff preview, report, fluxo base→Variant, sample, UPM | ✅ |
| **M5** | Piloto: 1 jogo, 1 tela real, 1 designer parceiro | ⬜ depende de pessoas e de um arquivo real |

### Desvios do plano original, e por quê

**`SPACE_BETWEEN` não gera espaçadores.** O plano previa inserir GameObjects espaçadores para
reproduzir o comportamento exatamente. Na implementação isso se mostrou uma má troca: um
espaçador é um GameObject sem node de origem, e a reconciliação assume que todo objeto no
prefab base tem um `FigmaNodeRef` para ser reencontrado. Daria para contornar com ids
sintéticos (`{idDoPai}#spacer0`), mas complica a parte mais delicada do sistema para atender
um recurso que o próprio linter do Figma já desencoraja. Ficou como aproximação reportada.

**O prefab base não tem Canvas.** A raiz gerada é um `RectTransform` esticado, e a resolução
de design fica em `UIViewRefs.DesignResolution`. Em jogo as telas normalmente vivem sob um
Canvas compartilhado; um Canvas por tela criaria batches separados e conflito de ordenação.
O componente `Screen` do kit existe para teste isolado.

**Binding sem codegen.** Confirmado como a escolha certa na prática: gerar uma classe de view
tipada obrigaria o import a esperar recompilação de assembly, transformando um import de 2
segundos num ciclo de 30.

**Definition of done do MVP:** o designer exporta uma tela real do Figma Web; o dev importa;
a tela abre correta em 1080×1920, 1920×1080 e 1280×800; textos com fonte, tamanho e
alinhamento certos; botões vindos dos prefabs do jogo e clicáveis; o dev adiciona um script
no Variant, o designer re-exporta com mudanças, e o script **continua lá**. Ciclo < 2 min.
