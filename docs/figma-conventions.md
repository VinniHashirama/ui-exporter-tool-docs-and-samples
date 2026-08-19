# Convenções do Figma

Guia para quem monta as telas. Seguir isso é o que faz a tela **virar interface de verdade
na Unity** em vez de virar um monte de retângulos sem comportamento.

O plugin valida cada regra daqui antes de exportar. Se apontar **erro**, o export é
bloqueado. Se apontar **aviso**, o export sai mas alguém vai ter que resolver depois — e
esse alguém provavelmente é você, uma semana depois, sem lembrar do contexto.

---

## A ideia em uma frase

**Você não está desenhando uma imagem da tela. Você está montando a tela.**

Cada botão no seu arquivo precisa ser uma instância do componente `Button/Primary` da
Library — não um retângulo com um texto em cima. Quando é uma instância, o importador troca
por um botão real do jogo, que já tem som de clique, animação de press e navegação por
gamepad. Quando é um retângulo, vira um retângulo.

É a mesma diferença entre escrever um texto usando estilos de parágrafo e escrever
formatando cada linha na mão. Funciona nos dois casos até a primeira vez que alguém precisa
mudar algo.

---

## 1. O frame da tela

O frame raiz precisa se chamar `screen/` + o nome da tela em PascalCase.

```
✅ screen/HomeMenu          → gera HomeMenu_Base.prefab
✅ screen/RewardPopup       → gera RewardPopup_Base.prefab
✅ screen/SettingsAudio     → gera SettingsAudio_Base.prefab

❌ Home Menu                → erro: fora do padrão
❌ screen/home menu         → erro: espaço não é permitido
❌ Frame 1                  → erro
```

O **tamanho** desse frame é a resolução de referência da tela. Combine com o time qual usar
e não varie entre telas do mesmo jogo — a Unity usa esse número para escalar tudo.

Exporte **um frame por vez**. Selecione o frame raiz e rode o plugin.

---

## 2. Use componentes da Library

Todo elemento interativo tem que ser uma instância de um componente do kit:

| Você quer | Use a instância de |
|---|---|
| Um botão com texto | `Button/Primary` ou `Button/Secondary` |
| Um botão só com ícone | `Button/Icon` |
| Uma janela / modal | `Window/Modal` |
| Um checkbox | `Toggle/Checkbox` |
| Um campo de texto | `InputField` |
| Uma barra de progresso / vida | `ProgressBar` |
| Uma lista que rola | `ScrollView` |
| Abas | `Tabs` |
| Um texto solto | `Label` |
| Um ícone solto | `Icon` |

A lista completa do que existe está em [`prefab-kit.md`](prefab-kit.md).

Se você precisa de algo que **não está no kit**, não improvise com retângulos: fale com o
dono da Library para o componente ser criado. Um componente novo no kit é meia hora de
trabalho; uma tela cheia de improviso é uma semana de retrabalho do dev.

Instância de componente que não existe no kit gera **aviso**, e na Unity aparece como caixa
genérica sem comportamento.

---

## 3. Prefixos e sufixos de nome

Quatro convenções, todas no nome da layer.

### `@Nome` — o dev precisa mexer nisso por código

Prefixe com `@` qualquer elemento que o código vai precisar acessar: botões que disparam
ação, textos que mudam em runtime, containers que recebem itens.

```
@PlayButton          → o dev acessa via view.Get<Button>("PlayButton")
@CoinLabel
@ItemContainer
```

Regras: PascalCase, sem espaços, **único dentro da tela** (nome repetido é erro). Se você
não marcar, o dev tem que caçar o objeto na hierarquia na mão — marque generosamente, marcar
demais não custa nada.

### `_nome` — ignore isso no export

Prefixe com `_` tudo que é andaime de design: anotações, réguas, especificações, versões
antigas, referências.

```
_notes
_specs
_old version
_measurements
```

Essas layers não vão para a Unity. Use à vontade — é assim que você mantém a documentação
no arquivo sem sujar o export.

### `nome#img` — achate isso em imagem

Sufixe com `#img` o que o exportador não sabe reconstruir: ilustrações, logos, vetores
complexos, qualquer coisa com gradiente, sombra, blur ou blend mode.

```
hero#img
logo#img
background-art#img
```

Essas layers são rasterizadas em PNG @2x e importadas como sprite. O visual sai **exatamente**
como você desenhou. O custo é que fica uma imagem — não escala nem muda de cor em runtime.

Se você usar gradiente/sombra/blur **sem** marcar `#img`, o plugin dá aviso e o efeito é
simplesmente perdido no import. Marque.

### `nome:loc.chave` — este texto é traduzido

Sufixe com `:` + a chave de localização.

```
title:loc.menu.title
button-label:loc.common.play
```

Por enquanto isso só é registrado no export — o sistema de tradução ainda não está ligado.
Mas marcar agora é de graça e evita ter que revisitar todas as telas depois.

---

## 4. Use Auto Layout

Auto Layout é o que faz a interface se adaptar. Sem ele, cada elemento fica numa posição
fixa e o texto em português (que é ~20% mais longo que o inglês) vaza do botão.

O que é traduzido para a Unity:

| Figma | Vira |
|---|---|
| Auto Layout horizontal / vertical | Layout equivalente na Unity |
| Espaçamento entre itens (`gap`) | Espaçamento |
| Padding | Padding |
| Alinhamento | Alinhamento |
| **Hug contents** | O container encolhe até o conteúdo |
| **Fill container** | O item ocupa o espaço disponível |
| **Fixed** | Tamanho literal |

Duas coisas que **não** têm equivalente direto e saem aproximadas (com aviso):

- **Space between** — é aproximado com espaçadores. Prefira `Fill container` em um dos itens.
- **Wrap** — vira uma grade de células iguais. Se os itens têm tamanhos diferentes, o
  resultado não vai bater.

---

## 5. Constraints, para o que não está em Auto Layout

Elementos posicionados livremente precisam de constraints, senão ficam grudados no canto
superior-esquerdo quando a tela muda de proporção. O jogo roda em celular retrato,
celular paisagem e desktop — a mesma tela vai esticar.

| Constraint | Comportamento |
|---|---|
| `Left` / `Top` | Cola na borda de origem |
| `Right` / `Bottom` | Cola na borda oposta |
| `Center` | Fica centralizado |
| `Left and right` / `Top and bottom` | Estica junto com o pai |
| `Scale` | Mantém a proporção da posição |

Regra prática: fundo e barras = esticar; HUD de canto = colar na borda; conteúdo
principal = centralizar.

### Safe area

Em mobile tem notch, câmera e barra de gestos. Coloque o conteúdo que não pode ser cortado
dentro de um frame chamado `SafeArea` — ele é respeitado automaticamente na Unity.

---

## 6. Cores e tipografia por token

Use os estilos da Library (`color/primary`, `text/h1`) em vez de valores soltos. Cor e fonte
fora de token geram aviso, e mais importante: sem token, trocar a paleta do jogo é editar
cada layer na mão.

---

## Checklist antes de exportar

- [ ] Frame raiz nomeado `screen/NomeDaTela`
- [ ] Todo interativo é instância de componente do kit
- [ ] O que o código precisa acessar está marcado com `@`
- [ ] Andaime de design está marcado com `_`
- [ ] Ilustrações e efeitos estão marcados com `#img`
- [ ] Auto Layout onde faz sentido
- [ ] Constraints no que está posicionado livremente
- [ ] Cores e textos usando estilos da Library
- [ ] O linter do plugin está sem erros

---

## Resumo das regras do linter

**Erro** — bloqueia o export:

| Regra | Problema |
|---|---|
| `root-frame-name` | Frame raiz fora do padrão `screen/<Nome>` |
| `duplicate-bind` | Dois `@Nome` iguais na mesma tela |
| `invalid-bind` | `@Nome` com caractere inválido |
| `empty-selection` | Nada selecionado, ou seleção não é um frame |

**Aviso** — passa, mas alguém paga depois:

| Regra | Problema |
|---|---|
| `default-layer-name` | Layer com nome automático (`Frame 42`, `Rectangle 5`) |
| `unknown-component` | Instância de componente que não está no kit |
| `unsupported-effect` | Gradiente, sombra, blur ou blend mode sem `#img` |
| `complex-vector` | Vetor complexo sem `#img` |
| `hardcoded-color` | Cor fora de token |
| `hardcoded-typography` | Estilo de texto fora de token |
| `space-between` | Aproximação imperfeita de `Space between` |
| `layout-wrap` | Aproximação imperfeita de `Wrap` |
| `rotated-node` | Node rotacionado — suporte limitado |
| `detached-instance` | Frame com nome de componente do kit, mas que não é instância |
| `mixed-text-styles` | Um texto com mais de uma formatação; vale a do primeiro trecho |
| `unsupported-text-case` | Small caps não existe no TextMeshPro; sai como maiúscula |
| `multiple-fills` | Fills empilhados; só o de cima é exportado |
| `mixed-fills` | Fills mistos: nenhum fundo é exportado |
| `empty-screen` | A tela não tem nenhuma layer dentro |

Uma tela bem montada exporta com **zero avisos** — é esse o alvo, não "poucos avisos".
