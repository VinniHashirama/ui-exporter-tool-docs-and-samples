# Kit de prefabs

Vocabulário canônico de UI. Os **nomes** são compartilhados por todos os jogos; os **prefabs
por trás deles** são de cada jogo. É isso que permite a mesma tela do Figma virar interface
com a identidade visual correta em jogos diferentes.

Público: dev que monta o kit, artista que entrega a arte.

---

Os dois lados do kit são **gerados por código**, e é isso que garante que os nomes canônicos
casem:

| Lado | Como gerar |
|---|---|
| Figma | Plugin → aba **Criar kit** → `Criar kit nesta página`, ou **Criar** por componente |
| Unity | `Window → Arvore → UI Exporter → Gerar kit placeholder` |

`tests/kit.test.ts` no repositório do plugin verifica que as listas não divergiram.

---

## De onde vem a aparência

Há dois caminhos, e a diferença decide o que o dev vê na tela importada.

**Padrão — a arte é do jogo.** O prefab do kit é montado na Unity pelo time do jogo, e a tela
exportada só instancia ele. É o que permite a mesma tela virar interface com a identidade
visual de jogos diferentes.

**Autorado no Figma.** O designer monta o componente no Figma e exporta um `.uikit`; o
importador gera o prefab com a arte, o tamanho, o 9-slice e a tipografia do design, e monta o
comportamento por cima. É o caminho de quem não quer esperar o prefab do jogo existir para ver
a tela de pé.

Os dois convivem: cada `canonicalName` segue um ou outro. O que **não** existe é o meio-termo
— num componente autorado no Figma, a aparência vem de lá e ajuste feito na mão é sobrescrito
no próximo import. Quem quiser um visual que o Figma não dita aponta um prefab próprio pela
`UIMappingTable`.

### Skinável × estrutural

| | Componentes | Por quê |
|---|---|---|
| **Skinável** | `Button/*`, `Label`, `Icon`, `Image`, `Panel` | O visual é superfície e conteúdo: dá para trocar a pele sem tocar no comportamento |
| **Estrutural** | `Slider`, `ScrollView`, `InputField`, `ProgressBar`, `Tabs`, `Window/Modal`, `Toggle/Checkbox` | A geometria **é** o comportamento |

Nos estruturais o componente não pode ser reconstruído a partir do desenho. `Slider` exige
`fillRect` e `handleRect` com âncoras específicas; `ScrollRect` exige um viewport com máscara
e um content com `ContentSizeFitter`. O desenho do Figma tem `Background`, `Fill` e `Handle`
posicionados livremente, e aplicar as constraints do design por cima escreveria justamente as
âncoras de que o `Slider` depende — o resultado brigaria consigo mesmo em runtime.

Esses continuam vindo do prefab do jogo. Autorá-los no Figma é v2, e exigirá escrever
*tokens* sobre o esqueleto existente em vez de reconstruir a hierarquia.

## Como o mapeamento funciona

Cada prefab do kit carrega um componente `UIKitComponent`:

```csharp
[canonicalName]  "Button/Primary"
[slots]          label → Transform, iconLeft → Transform, iconRight → Transform
```

O importador **varre o projeto** procurando prefabs com `UIKitComponent` e monta a tabela
sozinho. Não há configuração manual: colocar o prefab no projeto com o `canonicalName` certo
já o registra.

A `UIMappingTable` (ScriptableObject, um por jogo) existe apenas para exceções: apontar um
`canonicalName` para um prefab específico quando há mais de um candidato, ou redirecionar um
nome que aquele jogo resolve de outro jeito.

**Slots são declarados pelo prefab, não adivinhados pelo builder.** O builder pergunta ao
prefab "onde vai o label?" e recebe um `Transform`. Isso significa que a hierarquia interna
do prefab é livre — você pode reestruturar um botão sem quebrar nenhuma tela já importada.

---

## Regras para todo prefab do kit

1. Raiz com `RectTransform` + `UIKitComponent`.
2. `canonicalName` exatamente igual ao da tabela abaixo — case-sensitive.
3. Todo slot da spec tem que existir. Slot não usado numa tela é desativado pelo builder.
4. Estados (`Default`/`Hover`/`Pressed`/`Disabled`/`Selected`) resolvidos **dentro** do
   prefab — via `Selectable.transition` ou animator. O exportador não manda estado no MVP.
5. Funciona em qualquer tamanho: 9-slice nos fundos, Auto Layout interno onde couber.
6. Nada de referência a cena, singleton ou manager. O prefab é autocontido.
7. Texto sempre `TextMeshProUGUI`, nunca `Text`.

---

## MVP: 15 componentes

### Containers

| `canonicalName` | Slots | Estrutura Unity |
|---|---|---|
| `Screen` | `content` | `Canvas` + `CanvasScaler` + `GraphicRaycaster`, filho `SafeArea` |
| `Panel` | `content` | `Image` (9-slice) + `RectMask2D` opcional |
| `Window/Modal` | `title`, `body`, `footer`, `close` | `Image` fundo, header com título + botão fechar, body, footer |
| `ScrollView` | `content` | `ScrollRect` + `Viewport` (`RectMask2D`) + `Content` (LayoutGroup + `ContentSizeFitter`) + `Scrollbar` |

`Screen` é especial: é o prefab do frame raiz. `canvas.width/height` do IR vira
`CanvasScaler.referenceResolution`, com `screenMatchMode = MatchWidthOrHeight`.

### Ações

| `canonicalName` | Slots | Estrutura Unity |
|---|---|---|
| `Button/Primary` | `label`, `iconLeft`, `iconRight` | `Image` (9-slice) + `Button` + `HorizontalLayoutGroup` |
| `Button/Secondary` | `label`, `iconLeft`, `iconRight` | idem, skin secundária |
| `Button/Icon` | `icon` | `Image` + `Button`, proporção quadrada |

Os três precisam dos 5 estados funcionando. `label` recebe o texto da instância do Figma
(propriedade `label`, ou o primeiro filho de texto); ícones sem valor são desativados.

### Entrada

| `canonicalName` | Slots | Estrutura Unity |
|---|---|---|
| `Toggle/Checkbox` | `label`, `checkmark` | `Toggle` + `Image` box + `Image` check (graphic do Toggle) |
| `Slider` | `fill`, `handle` | `Slider` + `Background` + `Fill Area/Fill` + `Handle Slide Area/Handle` |
| `InputField` | `text`, `placeholder` | `TMP_InputField` + `Image` fundo + `RectMask2D` no viewport |

### Exibição

| `canonicalName` | Slots | Estrutura Unity |
|---|---|---|
| `Label` | — | `TextMeshProUGUI` puro |
| `Icon` | — | `Image`, `preserveAspect = true` |
| `Image` | — | `Image`, `type` conforme `fill.scaleMode` |
| `ProgressBar` | `fill`, `label` | `Image` fundo + `Image` fill (`type = Filled`) + label opcional |

`Label` é o destino de todo node `kind: "text"` que não está dentro de uma instância. Os
níveis tipográficos (`h1`…`caption`) **não** são prefabs separados — vêm de `text.token` +
`FontMap`.

### Navegação

| `canonicalName` | Slots | Estrutura Unity |
|---|---|---|
| `Tabs` | `tabContainer`, `indicator` | `HorizontalLayoutGroup` + `ToggleGroup`, filhos `Toggle` |

---

## Backlog (v0.2+)

**Entrada:** `Toggle/Switch`, `RadioGroup`, `Dropdown`, `Stepper`
**Exibição:** `Badge`, `Tag/Chip`, `Avatar`, `Tooltip`, `Divider`
**Feedback:** `Toast`, `Dialog/Confirm`, `Spinner`, `EmptyState`
**Navegação:** `NavBar`, `BottomBar`, `Pagination`
**Jogo:** `HUD/StatBar`, `Currency`, `ItemSlot`, `Card`, `Reward`

Um nome só entra nesta lista quando aparece em duas telas de jogos diferentes. Componente
canônico que existe para um caso único é custo de manutenção sem retorno.

---

## Kit placeholder (o que o MVP usa)

O MVP usa prefabs **wireframe** para provar o pipeline sem depender de produção de arte:

- fundos: sprite branco 9-slice de 16×16 com bordas de 4px, tingido por `Image.color`
- paleta: cinzas + um accent, o suficiente para distinguir hierarquia
- texto: fonte default do TMP
- estados: `Selectable.transition = ColorTint`
- ícones: um sprite genérico de placeholder

Trocar por arte real depois é **editar os prefabs do kit** — nenhuma tela precisa ser
re-exportada nem re-importada, porque as telas referenciam os prefabs por instância aninhada.
É o ponto principal de fazer o kit placeholder primeiro: desacopla a prova técnica do
cronograma de arte.

## Spec de entrega de arte (quando a arte real entrar)

- **Fundos 9-slice:** PNG com as 4 bordas identificáveis; entregar as medidas de borda em px.
  Área central mínima de 2×2px. Vindo do Figma, a borda é derivada do raio dos cantos — não
  precisa entregar medida nenhuma.
- **Dimensão:** múltiplo de 4 em ambos os lados. É o que a compressão em blocos (ASTC, DXT,
  ETC) exige; sem isso a textura fica em RGBA32 e ocupa várias vezes mais memória. Potência de
  2 não é necessária para UI. O importador relata quem não atende, mas não corrige.
- **Ícones:** quadrados, tamanho base 64×64 @1x, exportar @2x. Padding interno de 4px @1x
  para não colar na borda.
- **Estados:** entregar as cores dos 5 estados por componente; sprite diferente por estado
  só quando ColorTint não resolve.
- **Nomenclatura:** `<component>_<parte>_<estado>.png`, ex. `button-primary_bg_pressed.png`.
- **Formato:** PNG 32-bit, sem premultiply.
