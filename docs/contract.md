# O contrato: UIIR

Spec técnica da fronteira entre o plugin do Figma e o importador da Unity. Público: devs.
Para as regras que o designer segue, ver [`figma-conventions.md`](figma-conventions.md).

O schema formal vive no repositório do plugin: [`schema/uiir.schema.json`](https://github.com/VinniHashirama/ui-exporter-tool-figma-plugin/blob/main/schema/uiir.schema.json) — em caso de
divergência, **o schema manda**. Este doc explica o *porquê* de cada decisão.

---

## Os dois pacotes

**Tela** — um zip com nome `<NomeDaTela>.uiexport`:

```
HomeMenu.uiexport
├─ ui.json              # o UIIR, valida contra schema/uiir.schema.json
└─ images/
   ├─ hero@2x.png
   └─ logo@2x.png
```

**Componente do kit** — um zip com nome `<NomeCanonico>.uikit`:

```
Button_Primary.uikit
├─ kit.json             # o mesmo UIIR, com o bloco `kit` presente
└─ images/
   └─ button-primary_bg_default@2x.png
```

Sem outras entradas. O importador **rejeita** o pacote inteiro se encontrar qualquer coisa
fora desse formato — ver [Segurança](#segurança).

O discriminador é o **nome da entrada JSON**, não o conteúdo: `ui.json` é tela, `kit.json` é
componente. Um pacote com as duas é recusado. Decidir pelo conteúdo — "tem bloco `kit`, então
é componente" — faria um pacote de componente com um campo faltando ser importado como tela,
que é exatamente o tipo de erro silencioso que este contrato evita.

O importador roteia pela extensão do arquivo, então um `.uikit` nunca é aberto como tela por
engano.

---

## O bloco `kit`

Presente só em `kit.json`. É o que faz o importador gerar um **prefab do kit** em vez de uma
tela.

```jsonc
"kit": {
  "canonicalName": "Button/Primary",
  "role": "button",                    // button|toggle|container|display|icon|image
  "sourceVariantId": "12:34",
  "ignoredVariants": ["State=Pressed", "State=Disabled"],
  "slots": [ { "name": "label", "nodeId": "12:40" } ]
}
```

**`role` existe porque a forma não diz o que a coisa é.** Um retângulo com texto dentro pode
ser um botão ou um rótulo, e o Figma não distingue os dois. O papel é declarado, não
adivinhado: ele escolhe o esqueleto de comportamento que a Unity monta por cima da pele que
veio do design.

**`slots` aponta por `nodeId`, nunca por nome.** Designer renomeia layer o tempo todo; o id
sobrevive a isso. Os slots são descobertos de duas formas, nesta ordem: pelas propriedades de
componente que o Figma já declara, e pelo prefixo `$` no nome da layer (`$label`, `$icon`) —
que é o caminho de quem montou o componente à mão.

**`sourceVariantId` trava a variante de origem.** Cada variante do Figma tem ids de node
próprios. Exportar de outra variante depois trocaria **todos** os ids de uma vez, e a
reconciliação, sem reencontrar nada, recriaria o prefab inteiro — mudando os `fileID` que
todas as telas do jogo referenciam. O importador compara com o id da raiz do prefab atual e
**recusa** quando mudou.

### O que o prefab do kit ganha, e de quem

| Vem do Figma | Vem da Unity |
|---|---|
| sprite, cor, tamanho, raio | `Button`/`Toggle` e os 5 estados por `ColorTint` |
| Auto Layout, padding, espaçamento | `raycastTarget` na área clicável |
| fonte, tamanho e alinhamento de texto | slots declarados em `UIKitComponent` |
| bordas de 9-slice | |

### Titularidade: um prefab só, da ferramenta

```
Assets/UI/Generated/Kit/<Nome>.prefab    ← DA FERRAMENTA. Reconciliado a cada import.
```

Sem par base/variante, ao contrário das telas. Em tela o arranjo funciona porque a ferramenta
escreve layout e o dev escreve scripts — conjuntos disjuntos. Numa **skin** os dois
escreveriam as mesmas propriedades (sprite, cor, tamanho), e o override do artista mascararia
todo re-export seguinte, em silêncio, justamente no caso para o qual a feature existe.

Um prefab por nome canônico também elimina a ambiguidade de resolução por construção — e
ambiguidade é o que dispara o caminho destrutivo do importador.

**A regra prática:** ou a aparência de um componente vem do Figma, ou o prefab é do jogo.
Nunca as duas na mesma propriedade. Quem precisa de um visual que o Figma não dita aponta um
prefab próprio pela `UIMappingTable`, que existe exatamente para isso.

O que sobrevive a um re-import do prefab do kit:

| | |
|---|---|
| Componente que a ferramenta não gerencia (`AudioSource`, scripts do jogo) | **sobrevive** |
| Aparência (sprite, cor, opacidade, layout, texto) | **é revertida para o design** |

A reversão é intencional: sem ela, um botão que perdeu a transparência no Figma continuaria
transparente na Unity para sempre.

**Adoção.** Se já existe um prefab naquele nome que a ferramenta não gerou — o kit
placeholder, ou um prefab feito à mão —, o import **para** e pede confirmação explícita.
Adotar reconstrói o corpo; a referência do prefab e o rect da raiz sobrevivem, então as telas
continuam apontando para ele.

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

Versão atual: **1.1.0**. A 1.1 acrescentou o bloco `kit`, opcional — um pacote de tela 1.0
continua importando sem nenhuma diferença, e é por isso que a mudança é MINOR e não MAJOR.

- **PATCH** — correção de descrição, sem efeito em dado.
- **MINOR** — campo opcional novo, ou valor novo em enum tolerado por default.
- **MAJOR** — campo removido/renomeado, obrigatoriedade nova, mudança de semântica.

Recusar é deliberado: adivinhar a intenção de um pacote de outra major gera prefab
silenciosamente errado, que é muito pior que um erro de import.

---

## 9-slice

Sem borda, um fundo de botão achatado em PNG distorce ao esticar — os cantos arredondados
achatam. `asset.nineSlice` resolve isso, e é preenchido de duas formas:

| Fonte | Onde vale |
|---|---|
| Anotação `#9s(t,r,b,l)` no nome da layer | Sempre, tela ou componente |
| Derivado do raio dos cantos + espessura do traço | **Só no export de componente** |

A derivação não vale em tela de propósito: ali `#img` marca ilustração e logo, arte que
fatiada sairia deformada. Num componente, `#img` é a pele do botão — exatamente o caso em que
a borda tem que ser preservada.

A anotação aceita 1, 2, 3 ou 4 valores, na mesma lógica do CSS: `#9s(12)` é tudo igual,
`#9s(8,16)` é vertical/horizontal.

**Cantos não são arestas.** `cornerRadius` é `[TL, TR, BR, BL]` e `nineSlice` é
`[top, right, bottom, left]`: cada aresta encosta em dois cantos e recebe o maior dos dois.

As bordas são sempre limitadas para caber no sprite, deixando ao menos 1px de área
esticável por eixo — sem isso um botão em formato de pílula (raio = metade da altura) geraria
um sprite degenerado, e esse é o formato de botão mais comum que existe. Quando o limite
aperta uma borda anotada, o export avisa.

Do lado da Unity, uma borda só tem efeito com `Image.Type.Sliced`, que o importador
seleciona automaticamente quando o sprite tem borda.

---

## Dimensão de imagem

O importador **relata** e nunca altera o pixel.

O que de fato importa é a dimensão ser **múltipla de 4**: a compressão em blocos (ASTC, DXT,
ETC) só se aplica nessa condição, e sem ela a textura fica em RGBA32, várias vezes maior na
memória. **Potência de 2 quase nunca faz diferença para UI em UGUI** — é requisito de
formatos e plataformas antigos, e tratá-la como problema geraria aviso em quase todo sprite.

| Onde | O quê |
|---|---|
| Plugin, no export | `asset-not-multiple-of-4` e `asset-oversized` |
| Unity, no import | `sprite/size-policy`, com a dimensão que atenderia |

A política é configurável (`SpriteSizePolicy`): `MultipleOfFour` por padrão, `PowerOfTwo`
para quem tem uma exigência concreta, `None` para silenciar.

**Por que não corrigir automaticamente.** Corrigir exigiria preencher a textura com pixels
transparentes e recortar o sprite de volta ao desenho. O recorte depende de
`ISpriteEditorDataProvider`, que vive num pacote que este não quer impor a todo jogo que o
instale; e preencher sem recortar espreme o desenho — imperceptível num fundo de 200px, 11%
num ícone de 18px. Entre uma correção que às vezes deforma e um aviso preciso, o aviso é a
escolha honesta. A correção de verdade é o `SpriteAtlas`, que resolve compressão e batching
de uma vez.

---

## Fora do escopo do MVP

Achatar em PNG via `#img` (o designer decide):

- gradientes, sombras, blur, blend modes
- vetores complexos e operações booleanas

Não implementado:

- geração de `SpriteAtlas`
- correção automática de dimensão de textura — ver acima
- estados/variantes automáticos — só o default é exportado; os estados vêm do prefab
- animações e transições
- sistema de localização — `text.locKey` é só registrado
- codegen de view tipada — no MVP o binding é `UIViewRefs.Get<T>(key)`
- backend UI Toolkit
- múltiplos breakpoints por tela
- Figma REST API
- componentes cuja geometria **é** o comportamento (`Slider`, `ScrollView`, `InputField`,
  `ProgressBar`, `Tabs`) autorados no Figma — ver [`prefab-kit.md`](prefab-kit.md)

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
- aceita apenas `ui.json` **ou** `kit.json`, mais `images/*.png` — nunca as duas entradas JSON
  no mesmo pacote
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
| **M1** | Plugin Figma: traversal, mappers, lint, assets, zip | ✅ 119 testes, saída validada contra o schema |
| **M2** | Kit placeholder na Unity: 15 prefabs + `UIKitComponent`/slots | ✅ gerado por código, não commitado |
| **M3** | Importador: unzip, gate de schema, resolver, solver, builder | ✅ 72 testes EditMode |
| **M4** | Diff preview, report, fluxo base→Variant, sample, UPM | ✅ |
| **M4.5** | Autoria de componente no Figma → prefab de kit, 9-slice, dimensão de textura | ✅ |
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
