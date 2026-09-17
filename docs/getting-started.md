# Começando

Do zero até uma tela do Figma montada na Unity. Duas partes: a instalação, que se faz uma
vez, e o ciclo por tela, que é o que vai virar rotina.

---

## Instalação (uma vez)

### Lado dev — o plugin

```bash
cd figma-plugin
npm ci --ignore-scripts
npm run build          # gera dist/code.js e dist/ui.html
```

No **Figma desktop**: `Plugins` → `Development` → `Import plugin from manifest...` → escolha
`manifest.json` do repositório do plugin.

O plugin passa a aparecer em `Plugins` → `Development` → `Arvore UI Exporter`. Cada designer
que for exportar precisa fazer esse import uma vez, apontando para a mesma pasta (repositório
clonado, ou uma pasta compartilhada com o `dist/` já buildado).

### Lado dev — a Unity

1. Instale via **Package Manager**: `Window → Package Manager → + → Add package from git URL...`
   ```
   https://github.com/VinniHashirama/ui-exporter-tool-unity-package.git
   ```
   Não precisa travar numa tag — dá para usar a versão mais atual sem problema. Se preferir
   editar direto, o mesmo endereço vale como valor de `com.arvore.uiexporter` no
   `Packages/manifest.json`.

2. Se o projeto for novo: `Window` → `TextMeshPro` → `Import TMP Essential Resources`. Sem
   isso não existe fonte default e nenhum texto renderiza.

3. `Window` → `Arvore` → `UI Exporter` → **Gerar kit placeholder**. Cria os 15 prefabs
   canônicos em `Assets/UI/Generated/Kit`.

4. `Assets` → `Create` → `Arvore` → `UI Exporter` → **Import Settings**. Aponte
   `Rounded Sprite` para o sprite gerado em `Kit/Sprites`, e — importante em projeto grande —
   restrinja `Kit Search Folders` à pasta do kit, senão o importador varre todos os prefabs do
   projeto a cada import.

5. `Assets` → `Create` → `Arvore` → `UI Exporter` → **Font Map**. Mapeie cada família e estilo
   que o time usa no Figma para o `TMP_FontAsset` correspondente. Sem isso todo texto sai na
   fonte default, e o relatório avisa em cada import.

### Lado designer — o arquivo do kit

**Tem botão para isso.** No plugin, aba **Criar kit** → `Criar kit nesta página`.

Isso gera, numa página chamada `UI Kit`:

- os **14 componentes canônicos**, com variantes de estado nos botões e propriedades de
  componente (`label`, `iconLeft`, `iconRight`, `title`) já ligadas;
- os **estilos de cor e de texto** (`color/primary`, `text/h1`…), que é o que faz os avisos de
  `hardcoded-color` e `hardcoded-typography` não aparecerem;
- um frame **`screen/Exemplo`** de 1080×1920 com `SafeArea` dentro, para duplicar e renomear.

Rodar de novo é seguro: componente que já existe na página não é tocado.

Os nomes gerados são exatamente os que o importador da Unity procura — os dois lados são
gerados por código e há teste garantindo que as listas não divergem. É por isso que vale usar
o botão em vez de montar na mão: nome digitado errado só aparece como `unknown-component` no
import, longe de quem pode corrigir.

O que **não** é componente: `Screen`. Na Unity ele é o prefab de Canvas para teste isolado; no
Figma o equivalente é a convenção de nome do frame, e é por isso que o kit entrega o
`screen/Exemplo` como template em vez de um componente.

Precisa de algo que não está na lista? Fale com o dono da Library para criar o componente
canônico, e peça ao dev para criar o prefab equivalente na Unity com o mesmo nome. Improvisar
com retângulo gera aviso no linter e, na Unity, uma caixa sem comportamento.

> Sem plano pago você não consegue *publicar* essa página como Library. O kit funciona igual —
> só não propaga automaticamente para outros arquivos.

---

## O ciclo por tela

### Designer, no Figma

1. **Crie o frame raiz** com o nome `screen/NomeDaTela` — PascalCase, sem espaço.
   O tamanho desse frame é a resolução de referência. Combine um valor por orientação com o
   time e não varie entre telas do mesmo jogo.

2. **Monte com instâncias** dos componentes do kit. Todo botão é uma instância de
   `Button/Primary`, não um retângulo com texto em cima — é isso que faz o resultado na Unity
   ter som de clique, animação de press e navegação por gamepad já funcionando.

3. **Use Auto Layout** onde faz sentido: listas, HUDs, botões que crescem com o texto. É o que
   impede o texto em português de vazar da caixa.

4. **Ponha constraints** no que está posicionado livremente. Fundo e barras esticam, HUD de
   canto cola na borda, conteúdo principal centraliza.

5. **Marque os nomes**:
   - `@PlayButton` no que o código precisa acessar
   - `_notes` no que é andaime de design e não deve ir para a Unity
   - `hero#img` em ilustrações, logos, e qualquer coisa com gradiente, sombra ou blur
   - `title:loc.menu.title` em texto traduzível

6. **Rode o plugin**: selecione o frame raiz → `Plugins` → `Development` → `Arvore UI Exporter`.

7. **Leia o relatório.** Erro bloqueia o export e a mensagem diz o que corrigir. Aviso passa,
   mas alguém paga depois. Clicar num item leva a viewport até a layer.

8. **Exportar** → o navegador baixa `NomeDaTela.uiscreen`. Entregue esse arquivo ao dev, ou
   deixe na pasta combinada.

O alvo é exportar com **zero avisos**, não "poucos avisos". A referência completa das regras
está em [`figma-conventions.md`](figma-conventions.md).

### Dev, na Unity

9. `Window` → `Arvore` → `UI Exporter` → **Escolher...** → selecione o `.uiscreen`.

10. **Confira o diff.** A janela mostra o que vai ser criado, preservado e **removido**.
    Remoção é a única operação que destrói trabalho: se um objeto desapareceu do design, o
    GameObject vai embora e leva o que estava pendurado nele. Nada é escrito até você
    confirmar.

11. **Importar.** Se houver remoções, aparece uma confirmação extra.

12. **Leia o relatório.** Componente não mapeado, fonte faltando e aproximação de layout são
    casos em que o import teve sucesso mas o resultado não é o que o designer desenhou.

13. **Trabalhe no Variant**, em `Assets/UI/Screens/NomeDaTela.prefab`. Nunca no `_Base`.

14. **Ligue o código** pelos binds:

    ```csharp
    var view = screen.GetComponent<UIViewRefs>();

    view.Get<Button>("PlayButton").onClick.AddListener(StartGame);
    view.Get<TMP_Text>("CoinLabel").text = coins.ToString();
    ```

    Coloque a tela sob o Canvas do jogo e configure o `CanvasScaler` com
    `view.DesignResolution`.

Quando o designer mudar a tela e re-exportar, repita do passo 9. O prefab base é atualizado e
o que você fez no Variant continua lá.

---

## O ciclo por componente

Use quando o visual do botão deve vir do Figma, e não do prefab do jogo — tipicamente porque
ainda não existe prefab do jogo, e esperar por ele deixaria a tela importada como um monte de
caixas cinzas.

### Designer, no Figma

1. Monte o componente e transforme em **Component** (`Ctrl/Cmd + Alt + K`). Frame comum é
   recusado no export.
2. Nomeie com o nome canônico — `Button/Primary`. Com variantes, o nome vai no **Component
   Set**.
3. Marque as layers que recebem conteúdo com `$`: `$label`, `$icon`. Se o componente já usa
   propriedades de componente do Figma, elas são detectadas sozinhas.
4. Selecione o componente e abra a aba **Componente** do plugin. Confira o nome, o papel e os
   slots encontrados.
5. **Exportar** → `Button_Primary.uicomponent`. Mande para o dev.

### Dev, na Unity

6. `Window → Arvore → UI Exporter`, **Escolher...**, selecione o `.uicomponent`.
7. Confira o painel: nome canônico, papel, slots, e quantas telas instanciam esse componente.
8. Se já existir um prefab naquele nome que a ferramenta não gerou — o kit placeholder, por
   exemplo —, o import pede **adoção** explícita. Adotar reconstrói o corpo do prefab; a
   referência dele sobrevive, então as telas continuam apontando para ele, mas o texto que elas
   sobrescreviam nos slots volta ao default e vale re-importar essas telas depois.
9. **Importar componente**. O prefab aparece em `Assets/UI/Generated/Kit/`.

Daí em diante, aquele componente tem a aparência ditada pelo Figma: sprite, cor, tamanho, raio
e tipografia são reescritos a cada import. Comportamento que você pendurar no prefab — um
script, um `AudioSource` — sobrevive. Se precisar de um visual que o Figma não dita, aponte um
prefab próprio pela `UIMappingTable` em vez de editar este.

**Exporte sempre da mesma variante.** Trocar a variante de origem faz o importador recusar o
pacote: os ids de node mudariam todos de uma vez e o prefab teria que ser refeito do zero.

---

## Teste de fumaça em 10 minutos

Antes de envolver o time, vale provar o caminho todo com uma tela mínima:

1. No Figma, crie um frame `screen/Teste` de 1080×1920.
2. Dentro dele, um retângulo esticado (`constraints: left+right, top+bottom`) como fundo.
3. Um componente `Button/Primary` com o texto `Jogar`, nomeado `@PlayButton`.
4. Um texto `@TitleLabel` qualquer.
5. Rode o plugin, corrija o que o linter apontar, exporte.
6. Importe na Unity e confira: o fundo estica, o botão é instância do prefab do kit e é
   clicável, o `UIViewRefs` da raiz tem os dois binds.
7. Adicione um script no Variant, peça para o designer mover o botão e re-exportar, importe de
   novo — o script tem que continuar lá.

Se o passo 7 funcionar, o pipeline está de pé.

**Sem acesso ao Figma?** Dá para rodar os passos 6 e 7 usando o pacote de exemplo já
versionado, sem depender de ninguém:

```bash
bash tools~/unity-test.sh   # no repo do pacote: roda o pipeline contra Samples~/HomeMenu.uiscreen
```

---

## Quando algo dá errado

| Sintoma | Causa provável |
|---|---|
| O plugin não aparece no Figma | `npm run build` não rodou, ou o import apontou para o manifest errado |
| "Frame raiz fora do padrão" | O frame não se chama `screen/NomeDaTela` |
| Toda instância virou caixa vazia | O kit não foi gerado, ou `Kit Search Folders` não inclui a pasta dele |
| Todo texto na fonte errada | Font Map não configurado, ou a família do Figma não está mapeada |
| Nenhum texto renderiza | TMP Essential Resources não importados |
| Canto arredondado saiu quadrado | `Rounded Sprite` não configurado nas Import Settings |
| O trabalho do dev desapareceu | Estava no `_Base` em vez do Variant |
| "O pacote é da versão X" | Plugin e pacote da Unity em majors diferentes do contrato |
