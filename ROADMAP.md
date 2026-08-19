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
| **M5** | **Piloto: 1 jogo, 1 tela real, 1 designer parceiro** | ⬜ próximo passo |

### O que está validado, e como

| Item | Como foi verificado |
|---|---|
| Export de tela do Figma | **Manualmente, no Figma real** |
| Travessia, lint, convenções, tokens | 74 testes com mock da API do Figma; saída validada contra o schema |
| Import, reconciliação, layout, texto, segurança do zip | 55 testes EditMode no Unity 6000.3 |
| Trabalho do dev sobrevive a re-export | `Reimport_PreservesDevWorkInTheVariant` — o teste que decide o MVP |
| Sample reproduzível byte a byte | Dois `npm run sample` seguidos produzem o mesmo hash |

### O que ainda não foi validado

- **Botão de criar kit no Figma.** Foi adicionado depois da validação manual do export. As APIs
  do Figma que ele usa passaram por typecheck contra as tipagens oficiais, mas nunca rodaram.
  Quem rodar primeiro, conte o que aconteceu.
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

- **Detecção automática de 9-slice.** Hoje o campo existe no contrato mas só é preenchido por
  anotação explícita. Sem isso, janelas e botões achatados em `#img` distorcem ao escalar.
- **Estados e variantes.** `State=Disabled` no Figma hoje não chega ao prefab; o import avisa e
  monta no estado default.
- **`SPACE_BETWEEN` exato.** A aproximação atual joga o espaço para dentro dos itens. Dá para
  fazer exato com espaçadores de id sintético (`{idDoPai}#spacer0`), preservando a reconciliação.
- **Calibração de métricas de texto.** `lineSpacing` e `characterSpacing` do TMP não estão nas
  mesmas unidades do Figma; as fórmulas atuais derivam do `faceInfo` da fonte e precisam de uma
  cena de teste visual por família.
- **Gradientes.** Hoje viram cor chapada com aviso. Reconstruir exigiria material ou textura
  gerada.
- **Geração de `SpriteAtlas`** para reduzir batches em telas com muitos ícones.

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
