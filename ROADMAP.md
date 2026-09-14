# Roadmap

Estado real do projeto, sem otimismo. O que está verde tem teste ou validação manual atrás; o que
não tem, está marcado.

## Onde estamos

| # | Entrega | Estado |
|---|---|---|
| **M0** | Contrato: schema + docs | ✅ código pronto; **falta a revisão com o dono da Figma Library** |
| **M1** | Plugin do Figma | ✅ 74 testes; export **validado manualmente no Figma real** |
| **M2** | Kit de prefabs placeholder na Unity | ✅ 15 prefabs gerados por código |
| **M3** | Importador da Unity | ✅ 55 testes EditMode |
| **M4** | Diff, relatório, fluxo base→Variant, sample, distribuição UPM | ✅ |
| **M4.5** | **Autoria de componente no Figma → prefab de kit** | ✅ 24 testes novos |
| **M5** | **Piloto: 1 jogo, 1 tela real, 1 designer parceiro** | ⬜ próximo passo |

### M4.5 — o que entrou

Feito antes do piloto de propósito: levar o piloto com botões que viram caixa cinza
comprometeria a avaliação do designer parceiro, e a migração do kit custa zero agora que
nenhum jogo usa a ferramenta.

- **Criar componente avulso no Figma.** Botão `Criar` por componente, além do "criar tudo", e
  um formulário de componente próprio (nome + papel) que já nasce com as layers de slot
  nomeadas. A lista que o designer vê passou a ser derivada da mesma fonte que constrói — o
  `<ul>` hardcoded que podia divergir saiu.
- **Export de componente** (`.uikit`, `schemaVersion` 1.1.0) com slots, papel, variante de
  origem travada e nomes de asset seguindo a spec de entrega de arte.
- **Importador de componente** na Unity: um prefab por nome canônico, reconciliado, com
  `UIKitComponent`, slots ligados por `nodeId` e o esqueleto de comportamento por papel.
- **9-slice de verdade.** O campo existia no contrato e nunca era preenchido — e, mesmo
  preenchido, não teria efeito, porque o importador sempre usava `Image.Type.Simple`. Agora é
  derivado do raio, clampado para caber no sprite, e o tipo `Sliced` é selecionado.
- **Dimensão de textura** relatada nos dois lados.

### Bugs pré-existentes corrigidos junto

Todos no caminho que **destrói trabalho do dev**, e nenhum tinha teste:

- `ComponentResolver` varria os prefabs sem ordenar, então qual prefab um nome canônico
  resolvia dependia da ordem do `AssetDatabase` — e mudar de prefab recria as instâncias.
- Ambiguidade de `canonicalName` era aviso e o import seguia escolhendo arbitrariamente. Não
  existe valor seguro a devolver ali: agora bloqueia.
- O diff era calculado **antes** do resolver rodar, então troca de prefab escapava da única
  confirmação do sistema. O resolver foi movido para o `Prepare` e o diff passou a prever
  recriação.
- `raycastTarget` e remoção de `Image` eram aplicados sem olhar se havia um `Selectable`
  dependendo deles — latente hoje, fatal no modo de kit.

### O que está validado, e como

| Item | Como foi verificado |
|---|---|
| Export de tela do Figma | **Manualmente, no Figma real** |
| Travessia, lint, convenções, tokens | 119 testes com mock da API do Figma; saída validada contra o schema |
| Import, reconciliação, layout, texto, segurança do zip | 72 testes EditMode no Unity 6000.3 |
| Trabalho do dev sobrevive a re-export | `Reimport_PreservesDevWorkInTheVariant` — o teste que decide o MVP |
| Import de componente ponta a ponta | `KitImportTests`, com o `.uikit` de sample |
| Botão importado é clicável | `Import_ButtonKeepsItsClickableArea` — o hazard que nenhum teste de asset pega |
| Sample reproduzível byte a byte | Dois `npm run sample` seguidos produzem o mesmo hash |

### O que ainda não foi validado

- **Botão de criar kit no Figma, e a criação de componente avulso.** As APIs do Figma que eles
  usam passaram por typecheck contra as tipagens oficiais, mas nunca rodaram. Quem rodar
  primeiro, conte o que aconteceu. As falhas silenciosas de estilo foram removidas — o que der
  errado agora aparece na lista de avisos do plugin em vez de virar um kit meio montado sem
  explicação.
- **Export de componente no Figma real.** O caminho é exercitado por 17 testes com o mock e
  pelo sample, mas nunca saiu de um arquivo de verdade. A parte mais provável de surpreender é
  a detecção de slots por `componentPropertyReferences`, que depende de como o Figma sufixa os
  ids das propriedades.
- **Ordem do array `fills`.** O plugin assume que o último elemento é o de cima. Se aparecer cor
  invertida em layer com fills empilhados, é isso — está marcado no código, e o linter já avisa
  quando há mais de um fill.
- **`figma.fileKey`.** Pode vir indefinido em alguns contextos; há fallback, mas não foi
  exercitado.
- **Instalação do pacote via URL de git com tag.** Os testes rodam com o pacote local (`file:`).
  O caminho que os jogos vão usar é o git, e só uma instalação de verdade confirma.

## Próximo: melhorar o export

Tema priorizado depois do primeiro teste real. Candidatos, em ordem aproximada de valor por
esforço:

- **Geração de `SpriteAtlas`.** Subiu para o topo: resolve de uma vez a compressão de textura
  (hoje só relatada) e o batching em telas com muitos ícones. É a correção de verdade para o
  problema de dimensão que o importador hoje apenas aponta.
- **Estados e variantes.** `State=Disabled` no Figma continua não chegando ao prefab; o export
  manda só o default e os estados vêm do tint. Sprite por estado exigiria carregar as quatro
  variantes no pacote.
- **Componentes estruturais autorados no Figma.** `Slider`, `ScrollView`, `InputField`,
  `ProgressBar` e `Tabs` — onde a geometria é o comportamento e o import precisaria escrever
  tokens sobre o esqueleto em vez de reconstruir a hierarquia.
- **`SPACE_BETWEEN` exato.** A aproximação atual joga o espaço para dentro dos itens. Dá para
  fazer exato com espaçadores de id sintético (`{idDoPai}#spacer0`), preservando a reconciliação.
- **Calibração de métricas de texto.** `lineSpacing` e `characterSpacing` do TMP não estão nas
  mesmas unidades do Figma; as fórmulas atuais derivam do `faceInfo` da fonte e precisam de uma
  cena de teste visual por família.
- **Gradientes.** Hoje viram cor chapada com aviso. Reconstruir exigiria material ou textura
  gerada.

## Depois

- **Codegen de view tipada.** `view.PlayButton` em vez de `view.Get<Button>("PlayButton")`. Fica
  para depois porque obrigaria o import a esperar recompilação de assembly, transformando um
  import de 2 segundos num ciclo de 30.
- **Backend UI Toolkit.** O IR é neutro de propósito; dá para adicionar um segundo renderizador
  sem reescrever o parser do Figma.
- **Múltiplos breakpoints por tela.** Hoje uma tela é um frame com uma resolução de referência.
- **Localização de verdade.** `text.locKey` já é registrado no IR, mas nada consome.
- **Figma REST API.** Deixaria o dev puxar sem depender do designer exportar. Traz um token para
  gerenciar, o que hoje o pipeline não tem — a decisão de manter zero segredos foi deliberada.
- **Distribuição por registry (OpenUPM ou registry próprio).** Daria notificação de update de
  verdade no Package Manager, que pacote de git não tem.

## Riscos abertos

| Risco | Estado |
|---|---|
| **Adoção** — designers não seguirem as convenções | Mitigado em parte: o kit agora é gerado por botão, e o linter bloqueia export fora do padrão. O teste real é o piloto |
| Nem Figma Library nem kit de prefabs canonizados no time | O código dos dois lados existe; falta a decisão organizacional de qual biblioteca é a oficial |
| Sem plano pago no Figma não há Library publicada | O kit funciona, mas não propaga entre arquivos. Ver [getting-started.md](docs/getting-started.md) |
| Divergência entre o kit do Figma e o da Unity | Coberto por teste (`kit.test.ts`) que compara as duas listas |
| Escopo inflar para "pixel-perfect" | A lista de fora-de-escopo em [contract.md](docs/contract.md) é contrato, não sugestão |

## Definition of done do MVP

O designer exporta uma tela real do Figma; o dev importa; a tela abre correta em 1080×1920,
1920×1080 e 1280×800; textos com fonte, tamanho e alinhamento certos; botões vindos dos prefabs do
jogo e clicáveis; o dev adiciona um script no Variant, o designer re-exporta com mudanças, e o
script continua lá. Ciclo completo em menos de 2 minutos.

Falta apenas fazer isso com uma tela real de um jogo real — que é o M5.
