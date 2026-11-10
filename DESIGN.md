# Design — Minhas Finanças

Referência visual do app. O design é o mesmo do app de treino (repositório `gym`): mesmos tokens, fontes e componentes, para os dois parecerem da mesma família. Todo o CSS está no `<style>` de `index.html`, e os tokens ficam em `:root`. Qualquer mudança de layout deve seguir este documento. Se surgir um padrão novo, atualize-o aqui.

## Princípios

1. **Feito para lançar na hora.** Uso com uma mão, no caixa ou na fila, às vezes com pressa. Alvos de toque grandes, valores grandes, poucas decisões por tela.
2. **Mobile-first.** O layout base é o do celular. O desktop é uma adaptação dentro de um único breakpoint (`min-width: 960px`).
3. **Tema escuro, sempre.** Não existe tema claro (`color-scheme: dark`). Não crie variações claras.
4. **Uma ação principal por tela.** Ela usa o laranja de destaque. O resto é neutro.
5. **Os números são os protagonistas.** Valores em R$, totais e porcentagens usam a fonte condensada e algarismos tabulares.

## Cores

Use sempre as variáveis. **Não escreva hex novo no CSS.** Se faltar uma cor, crie um token em `:root`.

### Superfícies e bordas

| Token | Valor | Uso |
| --- | --- | --- |
| `--bg` | `#0E0F11` | Fundo da página (igual ao `theme-color`) |
| `--superficie` | `#17191C` | Cards, sheets, segmentos |
| `--superficie-2` | `#202328` | Inputs, opções dentro de sheets, menus, hover de ícone |
| `--nav` | `#141619` | Barra inferior no celular |
| `--borda-suave` | `#22252A` | Divisórias entre itens de lista |
| `--borda` | `#2A2E34` | Borda padrão de cards e inputs |
| `--borda-forte` | `#3A3F47` | Botão tracejado, chips, selo de somente leitura |
| `--lateral` | `#111316` | Barra lateral no desktop |
| `--lateral-borda` | `#1E2125` | Borda da barra lateral no desktop |

### Texto

| Token | Valor | Uso |
| --- | --- | --- |
| `--texto` | `#F2F3F5` | Texto principal e títulos |
| `--texto-2` | `#C9CDD3` | Texto secundário com destaque |
| `--suave` | `#9BA1AA` | Labels, legendas, ícones inativos |
| `--apagado` | `#6B717A` | Placeholders, numeração, dias futuros |

### Semânticas

| Token | Valor | Significa |
| --- | --- | --- |
| `--destaque` | `#FF6A3D` | **Agora / ação**: botão primário, aba ativa, conta que vence hoje, foco |
| `--destaque-hover` | `#FF7D55` | Hover do botão primário |
| `--destaque-tinta` | `#1B0D06` | Texto e ícone sobre `--destaque` |
| `--destaque-fundo` | `#2A1A13` | Fundo de item selecionado ou ativo |
| `--ok` | `#4CC38A` | **Feito / positivo**: conta paga, receita, sobra no mês, aviso de sucesso |
| `--ok-fundo` | `rgba(76,195,138,.1)` | Fundo de linha concluída |
| `--desce` | `#FF8A63` | Alertas leves (gasto acima da meta do grupo) |
| `--erro` | `#E5484D` | Erros, conta atrasada, mês no negativo e hover de ações destrutivas |

**Regra:** laranja indica o que fazer agora (pagar, lançar), verde indica o que já foi pago ou é positivo, vermelho indica atraso ou falta de dinheiro. Não troque esses papéis.

### Gráfico

Quando os gráficos chegarem, as séries usam, em ordem: `#FF6A3D`, `#5AB8E8`, `#E8C35A`, `#4CC38A`, `#B497FF`, `#FF8FB8` (constante `CORES` no JS, igual ao gym). Eixos, grade e tooltip usam cores escritas à mão que espelham os tokens de superfície e texto (o Chart.js não lê variáveis CSS). Se mudar um token, atualize também os valores escritos no `desenharGrafico()`.

## Tipografia

| Token | Fonte | Uso |
| --- | --- | --- |
| `--fonte` | Barlow 400–700 | Texto corrido, labels, botões comuns |
| `--fonte-titulo` | Barlow Condensed 500–700 | Títulos, botão primário, **todos os números e valores em R$** |

A base é `16px / 1.45`. **Inputs nunca ficam abaixo de 16px**, para o iOS não dar zoom.

| Elemento | Tamanho | Fonte / peso |
| --- | --- | --- |
| Título de página (`.pagina-topo h1`, `.titulo-pagina`) | 34px (`line-height: 1`) | Condensada 700 |
| Mês no seletor (`.seletor-mes h1`) | 34px, com o ano em sobretítulo | Condensada 700 |
| Título de sheet | 28px | Condensada 700 |
| Título de card | 26px | Condensada 700 |
| Valor de estatística (`.stat b`) | 30px | Condensada 700 |
| Valor no lançamento rápido | 26px ou mais | Condensada 600 |
| Botão primário | 20px (22px no `.grande`) | Condensada 700 |
| Texto padrão | 16px | Barlow 400 |
| Botão comum | 15px | Barlow 600 |
| Label de campo | 13px (`.8125rem`) | Barlow 600, `--suave` |
| Sobretítulo (`.sobretitulo`) | 13px, MAIÚSCULAS, `letter-spacing: .08em` | Barlow 700, `--suave` |
| Legenda / meta | 12–14px | Barlow, `--suave` |

Para números use a fonte condensada. A classe `.num-fonte` já inclui `tabular-nums`, para os valores de uma lista ficarem alinhados.

## Espaçamento, raios e tamanhos

**Espaçamento:** use valores da escala `4 · 8 · 12 · 16 · 20 · 24` (6, 10, 14 e 18 existem como ajustes finos). A margem lateral do conteúdo é **16px** no celular e **40px** no desktop.

**Raios:**

| Valor | Uso |
| --- | --- |
| `--raio-p` 12px | Inputs, segmentos, itens da barra lateral |
| 14px | Botões |
| `--raio` 16px | Opções, menus, estatísticas, linhas de lançamento |
| 20px | `.cartao` |
| 28px no topo (24px no desktop) | Sheets |
| Metade da altura | Pílulas, chips, avatar, FAB, selo |

**Alvos de toque (mínimos):**

| Altura | Elemento |
| --- | --- |
| 44px | Botões de ícone, setas do seletor de mês, pílula de data, avatar |
| 48px | Inputs, selects, botões comuns |
| 56px | Botão primário grande, FAB, itens de `.lista-simples` |

Não crie alvos de toque menores que 44px.

## Layout

| | Celular (< 960px) | Desktop (≥ 960px) |
| --- | --- | --- |
| Navegação | `.tabbar` fixa embaixo, 72px + safe area, 4 abas (*Mês*, *Lançamentos*, *Fixas*, *Evolução*) | Barra lateral de 248px com marca e as 4 abas |
| Conteúdo | Coluna de até 640px, centralizada | Até 1120px, alinhado à esquerda |
| Mês | Seletor de mês e avatar no topo | Igual; resumo e *A pagar* lado a lado (próximas etapas) |
| Lançar | FAB acima da barra inferior | FAB no canto inferior direito |
| Sheets | Sobem de baixo, com alça | Modal centralizado, sem alça |

- Tudo que fica fixo embaixo (FAB, aviso) se posiciona acima de `--altura-nav` + `--seguro-baixo`.
- Respeite `env(safe-area-inset-bottom)` via `--seguro-baixo` em qualquer novo elemento fixo.
- **Não crie outros breakpoints** sem necessidade clara. Ajustes de desktop vão dentro do `@media (min-width: 960px)` que já existe, no fim do `<style>`.

## Componentes

Reutilize estes componentes antes de criar um novo. Os marcados com *(gym)* ainda não estão no `index.html`: copie o CSS do gym quando forem usados pela primeira vez.

| Classe | O que é | Observações |
| --- | --- | --- |
| `.botao` | Botão base com contorno | 48px de altura |
| `.botao.primario` | Ação principal (laranja) | Só uma por tela ou contexto |
| `.botao.primario.grande` | Ação principal de largura total | Ex.: "Entrar", "Salvar lançamento" |
| `.icone` / `.icone.perigo` | Botão só com ícone, redondo | **Sempre com `aria-label`**. `.perigo` fica vermelho no hover |
| `.cartao` | Container de superfície | Use `.vazio` dentro dele para estados vazios |
| `.sobretitulo` | Rótulo de seção em maiúsculas | Também usado como `h2` de grupo |
| `.pagina-topo` | Cabeçalho de aba com `h1` | Pode conter segmento ou busca |
| `.topo-mes` / `.seletor-mes` | Cabeçalho da aba Mês: setas ‹ › com o mês no meio (ano em sobretítulo) e o avatar à direita | Setas são `.icone` com `aria-label` |
| `.conta` / `.avatar` / `.menu-conta` | Menu da conta: e-mail, selo de leitura e "Sair" | `aria-expanded` no avatar |
| `.selo-leitura` | Pílula "Somente leitura" no menu da conta | Aparece só em `modoLeitura` |
| `.so-dono` | Marca toda ação que altera dados | Fica `hidden` em `modoLeitura`. Não é um estilo, é uma regra |
| `#aviso` | Toast (verde para sucesso, `.erro` para vermelho) | Chamado via `aviso()` no JS; some em 3,5s |
| `.fab` *(gym)* | Botão flutuante de criar (claro sobre escuro) | "Lançar", canto inferior direito, ícone + texto |
| `.segmento` *(gym)* | Controle segmentado | Ex.: *Contas fixas* / *Parcelamentos*; grupo 50/30/20 com 3 opções |
| `.chip` *(gym)* | Filtro que liga e desliga | `aria-pressed` |
| `.opcao` *(gym)* | Escolha grande, estilo radio, em sheets | `aria-pressed` |
| `.lista-simples` *(gym)* | Lista tocável com divisórias | Itens de 56px |
| `.busca` *(gym)* | Input com ícone de lupa | `type="search"` + `aria-label` |
| `.data-pill` *(gym)* | Pílula com input de data invisível por cima | |
| `.stat` / `.resumo` *(gym)* | Estatística (rótulo + número grande) | 2 colunas, 4 no desktop |
| `dialog.sheet` *(gym)* | Bottom sheet / modal | Estrutura abaixo |

### Estrutura de um sheet

```html
<dialog id="dlg-x" class="sheet" aria-labelledby="titulo-x">
  <div class="sheet-alca" aria-hidden="true"></div>
  <div class="sheet-cabeca">
    <h2 id="titulo-x">Título</h2>
    <button class="icone" type="button" data-fechar aria-label="Fechar">…ícone x…</button>
  </div>
  <div class="sheet-corpo">…</div>
</dialog>
```

O sheet abre com `abrirDialogo('#dlg-x')`. Um toque no fundo escurecido ou em qualquer `[data-fechar]` fecha.

## Ícones

- SVG inline, `viewBox="0 0 24 24"`, `fill="none"`, `stroke="currentColor"`, `stroke-linecap/linejoin="round"`, `stroke-width` entre 2 e 2,8, sempre com `aria-hidden="true"`.
- Tamanhos: 14 e 16px (inline em texto), 18 e 20px (botões), 22px (navegação e setas do mês), 26px (marca).
- A marca é uma carteira (retângulo com uma faixa e um traço), no mesmo quadrado laranja do gym.
- No JS, use o objeto `ICONE` e o helper `svg()`. Ao criar um ícone novo que se repete, adicione-o ao `ICONE`.
- Não use biblioteca de ícones nem emoji como ícone.

## Movimento

- O sheet sobe com `.22s ease-out` e o aviso aparece em `.25s`.
- Animações curtas e funcionais, sem animação decorativa.
- `prefers-reduced-motion` desliga tudo. Não remova essa regra.

## Acessibilidade

- O foco é visível: `:focus-visible` com contorno laranja de 2px. Não remova outlines.
- Estado sempre via ARIA: `aria-current="page"` (aba), `aria-pressed` (chips, opções, segmentos), `aria-selected` (tabs), `aria-expanded` (menu).
- Todo input tem `label` ou `aria-label`. Todo botão de ícone tem `aria-label` descritivo, inclusive com o nome do item (ex.: "Marcar Netflix como paga").
- Use `.sr` para texto só para leitores de tela.
- Cor nunca é o único sinal: conta paga também tem o ícone de check, atrasada também diz "atrasada", e valor negativo também tem o sinal.

## Texto da interface

- Português do Brasil, tom direto e informal ("O que falta pagar", "Quanto foi?").
- Botões começam com verbo e são curtos: "Lançar", "Salvar lançamento", "Marcar como paga".
- Títulos e botões usam só a primeira letra maiúscula.
- Estados vazios dizem o que fazer: "Nenhum lançamento neste mês. Toque em “Lançar” para começar."
- Avisos de sucesso são curtos, com exclamação ("Lançamento salvo!"). Erros começam com "Não foi possível …" quando há contexto.
- Valores sempre em reais pelo helper de dinheiro (`R$ 1.234,56`); datas pelos helpers (`qui, 5 nov`, `Novembro 2026`).

## Checklist para mudanças visuais

- [ ] Usei tokens e componentes existentes (ou criei token novo em `:root`)
- [ ] Testei em ~375px e em ≥ 960px
- [ ] Alvos de toque com 44px ou mais, inputs com 16px ou mais
- [ ] Elementos fixos respeitam a barra inferior e a safe area
- [ ] Estados (ativo, feito, desabilitado, vazio, erro) estão cobertos
- [ ] ARIA e `aria-label` corretos, foco visível
- [ ] Ações que alteram dados têm `.so-dono` (somem no modo leitura)
- [ ] Atualizei este arquivo se criei um padrão novo
