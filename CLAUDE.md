# Minhas Finanças

App web para controlar as finanças pessoais: lançar gastos e receitas, ver o que falta pagar no mês e acompanhar o gasto pela regra 50/30/20. Substitui a planilha mensal do Google Drive. Uso pessoal, mobile-first, pensado para lançar um gasto no celular na hora em que ele acontece.

Segue o mesmo padrão do app de treino (repositório `gym`): mesma stack, mesmas convenções, mesmo design e o mesmo projeto Supabase.

- **Tudo está em `index.html`**: HTML, CSS e JavaScript (ES module). Não tem build, bundler, `package.json` nem código de servidor.
- **Backend:** Supabase (Auth com e-mail/senha e Google, banco Postgres com RLS), acessado direto do navegador pelo `supabase-js` v2. **O projeto Supabase é compartilhado** com o gym e com futuros sistemas pessoais; as tabelas deste app ficam no schema **`financas`**.
- **Gráfico:** Chart.js v4. As duas bibliotecas vêm do CDN jsDelivr, e as fontes (Barlow e Barlow Condensed) do Google Fonts.
- **Hospedagem:** Cloudflare Workers servindo arquivos estáticos (`wrangler.jsonc`), no domínio `account.julianodrebel.com.br`. O deploy é automático: o push para o GitHub dispara a esteira da Cloudflare. Nunca é feito da máquina local.
- O **design** está documentado em `DESIGN.md`. Leia antes de qualquer mudança visual.

## Comandos

```bash
npx serve .                 # rodar local (ou: python -m http.server 8000)
git switch develop          # base para qualquer trabalho novo
```

- Não existem testes automatizados, lint nem formatter. Para validar, sirva o app localmente e teste no navegador **em largura de celular (~375 px) e em desktop (≥ 960 px)**, com o console sem erros.
- Abrir o `index.html` direto pelo arquivo (`file://`) não funciona, porque o app usa ES modules.
- O login com Google só funciona localmente se a URL local estiver em **Supabase → Authentication → URL Configuration**. Para testar, entrar com e-mail e senha é mais simples.

## Estrutura do `index.html`

Não leia o arquivo inteiro quando ele crescer: localize a seção com busca (`grep -n`) e leia só o trecho necessário.

1. **`<head>`**: metas (`theme-color` igual a `--bg`), fontes e Chart.js.
2. **`<style>`**: tokens em `:root`, depois blocos marcados com `/* ---------- nome ---------- */` (campos e botões, login, estrutura do app, sheets, mês, lançamentos) e, no fim, `@media (min-width: 960px)` com todo o layout de desktop.
3. **Markup**: `#tela-login`, `#tela-app` (com `nav.tabbar` e `.conteudo`, onde cada aba é uma `section.aba#aba-<nome>`: `mes`, `lancamentos`, `fixas`, `evolucao`) e, no fim do `<body>`, os `<dialog class="sheet">`.
4. **`<script type="module">`**: blocos marcados com `// ---------- nome ----------`, nesta ordem: configuração do Supabase, utilidades (inclui dinheiro), sheets, autenticação, modo leitura, abas, mês, listas (meios de pagamento e categorias), lançamentos, histórico de descrições (autocompletar) e lançamento rápido.

Código novo vai no bloco correspondente. Se for um assunto novo, crie um bloco com o mesmo estilo de marcador, na posição que faça sentido.

## Convenções de código

- **Nomes em português**: variáveis, funções, IDs, classes CSS e comentários (`carregarLancamentos`, `#lista-lancamentos`, `.conta-fixa`). Siga isso.
- Estilo: 2 espaços, aspas simples, ponto e vírgula, `const` por padrão, arrow functions e template strings.
- `$` e `$$` são atalhos de `querySelector` e `querySelectorAll`.
- O estado fica em variáveis `let` no escopo do módulo (`mesAtual`, `modoLeitura`…). Não há framework nem store.
- **`carregarX()`** busca do Supabase e atualiza o estado. **`desenharX()`** renderiza a partir do estado.
- A renderização é feita com template string e `innerHTML`. **Todo texto vindo do usuário ou do banco passa por `esc()`.** Isso não é opcional, porque evita XSS.
- Eventos de listas dinâmicas usam delegação: um listener no container, `e.target.closest('.classe')` e `data-id` no botão.
- Chamadas ao Supabase seguem este padrão (o client já aponta para o schema `financas`):
  ```js
  const { data, error } = await supabase.from('tabela').select('...');
  if (error) return aviso(error.message, true);
  ```
  Ao salvar, desabilite o botão (`botao.disabled = true`) durante a chamada.
- Para feedback ao usuário use `aviso(msg)` (sucesso) ou `aviso(msg, true)` (erro). Não use `alert()`.
- Para dialogs use `abrirDialogo('#id')` e `fecharDialogo('#id')`. Botões com `data-fechar` fecham o dialog.
- Use os helpers que já existem em vez de criar novos:
  - Datas: `hoje()`, `isoLocal()`, `fmtData()`, `fmtDia()`. Não use `toISOString()` direto, porque ele usa UTC e muda o dia à noite no Brasil.
  - Meses: `inicioMes()`, `somarMeses()`, `nomeMes()`. Um mês é sempre guardado como o 1º dia (`'2026-11-01'`). O mês em que um lançamento é pago sai de `mesDoPagamento(data, meio)`.
  - Dinheiro: `centavos()` (valor do banco → centavos), `fmtReais()` (centavos → `R$ 59,90`), `lerReais()` (texto digitado → centavos ou `null`), `paraBanco()` e `valorCampo()`.
  - Ordenação e busca: `porNome` e `normalizar()`, que ignoram acentos e maiúsculas.
  - Ícones: o objeto `ICONE` e o helper `svg()`.
- **Dinheiro:** no banco é `numeric(12,2)`; no JS, some em **centavos inteiros** (`centavos()`) e mostre com `fmtReais()`. Nunca some valores em ponto flutuante direto.
- **Modo leitura:** toda ação que altera dados (botões, FAB, formulários) recebe a classe `.so-dono`, que some quando a pessoa logada é leitora (`modoLeitura`). O banco também recusa a escrita, então isso é só a interface.
- **Não adicione dependências, frameworks, etapas de build nem divida o arquivo** sem combinar antes. A simplicidade de um arquivo só é intencional.
- O texto da interface é em **português do Brasil**.

## Banco de dados (Supabase)

Schema **`financas`**, no mesmo projeto do gym. O client é criado com `{ db: { schema: 'financas' } }`.

| Tabela | Conteúdo |
| --- | --- |
| `leitores` | Quem pode ler os dados de quem (`dono_id`, `leitor_id`). Mantida pelo SQL Editor; o app só lê |
| `meios_pagamento` | Lista comum (sem `user_id`), mantida pelo SQL Editor; o app só lê. `tipo` (`cartao`, `pix`, `boleto`, `folha`, `beneficio`, `dinheiro`, `outro`); cartões têm `dia_fechamento` e `dia_vencimento`; `ativo = false` esconde |
| `categorias` | Lista comum, mantida pelo SQL Editor; o app só lê. `tipo` (`despesa`/`receita`) e `grupo_padrao` (grupo 50/30/20 sugerido) |
| `lancamentos` | Cada gasto ou receita. `tipo`, `data`, `mes` (1º dia do mês em que é pago), `descricao`, `valor` (despesa negativa = reembolso), `grupo` (`essencial`/`diversao`/`investimento`, obrigatório só em despesa), `categoria_id`, `meio_pagamento_id`, `pago`, `observacao` |

Tabelas planejadas para as próximas etapas: `contas_fixas` e `parcelamentos` (com `lancamentos.conta_fixa_id`, `parcelamento_id` e `parcela`).

- **Toda tabela nova precisa, no mesmo script:** `GRANT` para `authenticated` (e `service_role`), RLS e policy. Desde 30/10/2026 o Supabase não dá mais acesso automático às tabelas novas; sem o `GRANT` a API responde *permission denied* (42501). **Nunca dê acesso ao papel `anon`.**
- Tabelas com dados do usuário têm `user_id uuid not null default auth.uid()`. **Não envie `user_id` nos inserts.**
- Policies de leitura liberam o dono (`auth.uid() = user_id`) e quem está em `leitores` para aquele dono. Policies de escrita liberam só o dono.
- A chave em `index.html` é a *publishable* (pública por design). **Nunca coloque a chave `service_role` ou secret no código.**
- Em *Exposed schemas* do Supabase, `financas` fica **junto** com `public`. Tirar `public` derruba o gym.
- **Ao mudar o schema:**
  1. Atualize o script de criação no `README.md`.
  2. Adicione uma seção em *Upgrading an existing database* com o SQL de migração dentro de `begin; … commit;`, sem perder dados.
  3. Lembre o usuário de rodar o SQL no Supabase **antes** de publicar o novo `index.html`.

### Decisões

- **Schema próprio (`financas`)** em vez de tabelas em `public`, para não misturar com o gym e com futuros sistemas pessoais no mesmo projeto Supabase.
- **Sem "Criar conta"** no login: as contas são criadas pelo painel do Supabase e o cadastro de novos usuários fica desligado.
- **Naiara é leitora:** vê os dados do Juliano, sem lançar nem alterar (`leitores`). Se um dia precisar lançar, libera-se a escrita para quem está em `leitores`.
- **Mês de um lançamento = mês em que é pago.** Compra no cartão vai para o mês de vencimento da fatura, calculado pelos dias de fechamento e vencimento do cartão.
- **Categorias e meios de pagamento** são listas fixas mantidas pelo SQL Editor, sem tela. As metas 50/30/20 ficam numa constante no código.
- **Contas fixas e parcelas** do mês são criadas automaticamente como pendentes ao abrir o mês, sem duplicar (`unique`).
- **Reembolso** é lançamento com valor negativo na mesma categoria, como na planilha.
- **Cartão:** compra a partir do dia de fechamento vai para a próxima fatura; se o vencimento é antes (ou no dia) do fechamento, a fatura vence no mês seguinte ao fechamento. Sem os dois dias cadastrados, o mês é o da data. O mês pode ser trocado à mão no formulário.
- **"Já foi pago" no formulário:** compra no cartão começa como *a pagar* (a fatura ainda não foi paga); o resto começa pago se a data já passou. Depois que o usuário mexe na caixa, o app não muda mais.
- **Meio de pagamento é obrigatório** no lançamento e vem preenchido com o do lançamento mais recente. Categoria é opcional.
- **Autocompletar:** a descrição usa um `<datalist>` com as descrições já usadas. Escolher uma repete tipo, grupo, categoria e meio da última vez, e o valor só se o campo estiver vazio.

## Git e fluxo de trabalho

- `master` é produção. `develop` é integração (e é a branch padrão no GitHub).
- Trabalho novo sai de `develop` numa branch de feature e volta por pull request para `develop`.
- Release: merge de `develop` em `master` com a versão como mensagem (`1.0.0`, `1.0.1`…).
- **Push é deploy.** Cada push para o GitHub dispara a esteira da Cloudflare: `master` vai para produção e as outras branches geram versões de preview. Por isso, mudanças de schema precisam estar aplicadas no Supabase antes do push.
- Mensagens de commit em **inglês**, no formato Conventional Commits (`feat: …`, `fix: …`, `refactor: …`, `docs: …`).
- **Nunca commite** `client_secret*.json`, a planilha exportada (`*.xlsx`) nem o SQL gerado pela importação (`tools/saida/`): estão no `.gitignore`.
- Só faça commit, push ou merge quando o usuário pedir.

## Documentação

- `README.md` é em **inglês**. Ao criar ou mudar uma funcionalidade, atualize a lista *Features*. Ao mudar o schema, siga as regras da seção de banco acima.
- `DESIGN.md` é a referência visual. Se uma mudança criar um padrão novo (token, componente, regra), registre lá.
- Decisões importantes tomadas durante uma tarefa devem ser registradas neste arquivo.

## Pontos de atenção

- `onAuthStateChange` chama `mostrarTela` dentro de `setTimeout`, como a documentação do Supabase recomenda. Não remova.
- O diretório de assets é a raiz do repositório. O `.assetsignore` deixa só o `index.html` ir para o site; arquivo novo que não deva ser público precisa estar coberto por ele.
