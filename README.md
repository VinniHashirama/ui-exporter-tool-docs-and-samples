# Figma → Unity UI Exporter

Pipeline para o time de UI/UX montar interfaces no **Figma** e o time de engenharia importá-las
direto na **Unity (UGUI)**, sem remontar telas na mão.

```
Figma ──[plugin]──► MinhaTela.uiscreen    ──[pacote UPM]──► Prefab da tela
                └─► Button_Primary.uicomponent  ──[pacote UPM]──► Prefab do componente
```

📖 **[Guia rápido (instalação + fluxo de uso)](https://vinnihashirama.github.io/ui-exporter-tool-docs-and-samples/)**

Este repositório é o **hub**: documentação, roadmap e samples. O código vive em dois
repositórios separados, por público-alvo.

| Repositório | O que é | Quem usa |
|---|---|---|
| [ui-exporter-tool-figma-plugin](https://github.com/VinniHashirama/ui-exporter-tool-figma-plugin) | Plugin do Figma (TypeScript) + o schema do contrato | **Designers** e quem mexe no plugin |
| [ui-exporter-tool-unity-package](https://github.com/VinniHashirama/ui-exporter-tool-unity-package) | Pacote UPM `com.arvore.uiexporter` | Devs Unity |
| **este** | Docs, roadmap, samples | Todos |

---

## Por onde começar

**Sou designer e vou montar telas.**
→ [Instalar o plugin](https://github.com/VinniHashirama/ui-exporter-tool-figma-plugin#1-instalar-no-figma)
(não precisa de Node, terminal, nem conta paga), depois
[as convenções](docs/figma-conventions.md).

**Sou dev Unity e vou importar telas.**
→ [Instalar o pacote](https://github.com/VinniHashirama/ui-exporter-tool-unity-package#1-instalar)
via Package Manager, depois [getting-started.md](docs/getting-started.md).

**Vou mexer na ferramenta.**
→ [contract.md](docs/contract.md) para entender o formato que liga os dois lados, e o
[ROADMAP.md](ROADMAP.md) para saber o que está em aberto.

---

## Como funciona

O pipeline tem exatamente **dois artefatos** e **nenhum servidor**:

| Artefato | Onde roda | O que faz |
|---|---|---|
| Plugin | Figma desktop (TypeScript) | Lê a seleção, valida contra as convenções, gera o `.uiscreen` |
| Pacote | Unity 6 Editor (C#) | Lê o `.uiscreen` e monta ou atualiza o prefab |

A fronteira entre os dois é o **UIIR** — um JSON versionado, descrito em
[`schema/uiir.schema.json`](https://github.com/VinniHashirama/ui-exporter-tool-figma-plugin/blob/main/schema/uiir.schema.json).
Nenhum dos lados conhece o outro: o plugin não sabe o que é um `RectTransform`, o importador não
sabe o que é um node do Figma.

### Fluxo de trabalho

**Designer** monta a tela num frame `screen/NomeDaTela` usando instâncias dos componentes do kit,
marca `@` no que o código acessa, roda o plugin, zera os erros do linter e exporta.

**Dev** escolhe o `.uiscreen` na janela do importador, confere o diff, importa, e trabalha no
**Prefab Variant** — nunca no `_Base`.

### A regra de ouro

```
Assets/UI/Generated/<Tela>/<Tela>_Base.prefab   ← DA FERRAMENTA. Sobrescrito a cada import.
Assets/UI/Screens/<Tela>.prefab                 ← DO DEV. Prefab Variant. Nunca tocado.
```

O importador nunca recria os GameObjects do base: reencontra cada node pelo `FigmaNodeRef` e
reusa o objeto, preservando os `fileID` de que os overrides do Variant dependem. É isso que faz o
trabalho do dev sobreviver a um re-export do designer.

## Escopo

O export de **tela** cuida de estrutura, layout, texto e hierarquia; o visual vem dos prefabs do
kit de cada jogo. É por isso que o resultado usa os componentes reais do jogo, com áudio e
navegação já funcionando, em vez de uma casca visual sem comportamento.

O export de **componente** fecha o outro lado: o designer monta o botão no Figma e a Unity gera o
prefab com a arte, o tamanho, o 9-slice e a tipografia do design, montando o comportamento por
cima. Serve para não precisar esperar o prefab do jogo existir para ver a tela de pé.

A regra que liga os dois: **a aparência de um componente vem do Figma ou do jogo, nunca das duas
na mesma propriedade.**

Fora do escopo: gradientes, sombras, blur, blend modes e vetores complexos, que são achatados em
PNG pelo designer via a convenção `#img`; e os componentes cuja geometria *é* o comportamento
(`Slider`, `ScrollView`, `InputField`…), que continuam vindo do prefab do jogo. Lista completa em
[contract.md](docs/contract.md).

## Documentação

| Doc | Para quem |
|---|---|
| [getting-started.md](docs/getting-started.md) | Instalação e o ciclo por tela, ponta a ponta |
| [figma-conventions.md](docs/figma-conventions.md) | **Designers** — como montar arquivos que exportam |
| [contract.md](docs/contract.md) | Devs — spec do UIIR, versionamento, mapeamentos, segurança |
| [prefab-kit.md](docs/prefab-kit.md) | Devs e artistas — spec dos prefabs do kit |
| [ROADMAP.md](ROADMAP.md) | Status, o que foi validado, próximos passos |

## Samples

[`samples/`](samples/) tem os dois pacotes de exemplo, reproduzíveis byte a byte
(`npm run sample` no repositório do plugin): `HomeMenu.uiscreen`, uma tela, e
`Button_Primary.uicomponent`, um componente. Servem para exercitar o importador sem depender de um
export real, e são os mesmos arquivos que o pacote da Unity carrega em `Samples~/` para os testes.

> Ferramenta interna da Arvore.
