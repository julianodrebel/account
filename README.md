# 💰 Minhas Finanças

A lightweight, mobile-first personal finance tracker that runs entirely in the browser. It replaces a monthly spreadsheet: log expenses in a few taps, see what is still to be paid this month, and track spending against the 50/30/20 rule. Data lives in [Supabase](https://supabase.com), in the same project as the [workout tracker](https://github.com/julianodrebel/gym), under its own `financas` schema.

The whole app is a single `index.html` file: no build step, no bundler, no server code.

## Features

- **Accounts** — sign in with email and password or with Google (Supabase Auth). There is no sign-up screen: accounts are created in the Supabase dashboard.
- **Read-only sharing** — a second person can be given read access to the owner's data (`financas.leitores`). They see everything, but the app hides every action that changes data and the database refuses writes.
- **Quick entry** — the *Lançar* button opens a sheet to log an expense or income: amount (typed as `59,90` or `1.234,56`), description, date, payment method, 50/30/20 group (*Essencial*, *Diversão*, *Investimento*), category, whether it is already paid, refund (stored as a negative expense) and a note. Picking a description used before fills in its type, group, category, payment method and amount from the last time. Picking a category suggests its group.
- **Month it is paid in** — every entry belongs to the month it is paid in. For credit cards with closing and due days set, the app puts the purchase in its bill's due month (a purchase on 20 Nov with closing on the 10th and due on the 15th goes to December) and leaves it as *a pagar* (to pay); the month can also be picked by hand.
- **Entries** — the *Lançamentos* tab lists the selected month's entries grouped by day, with income, expenses and balance for what is shown. Search by description, category, payment method or note, and filter by group, income or entries still to pay. Tap an entry to edit or delete it.
- **Month overview** — the *Mês* tab shows income, expenses, what is left and what is still to pay. *Pendentes* lists unpaid bills and income not yet received by due date (late ones in red); credit card purchases are grouped into one line per card bill. One tap on the circle marks a bill, or a whole card bill, as paid. Tapping a card bill shows its entries; tapping any other line opens it to fix the amount. Next to it, the 50/30/20 rule: what went to *Essencial*, *Diversão* and *Investimento* as a share of the month's income (of the spending, when there is no income yet) against the 50% / 30% / 20% targets, and the month's spending per payment method.
- **Fixed bills** — in *Fixas → Contas fixas*, register bills and recurring income (Netflix, rent, salary) with amount, day of the month, payment method, group, category and first/last month. When a month from now on is opened for the first time, each fixed bill becomes a pending entry in it (only once: an entry deleted on purpose does not come back). Editing a fixed bill updates its unpaid entries from this month on.
- **Installment purchases** — in *Fixas → Parcelamentos*, register a purchase with the installment amount, number of installments, purchase date and payment method; all installments are created at once in the months they are paid in (card bills' due day). A purchase that started before the app can start from installment *n*. The list shows how many are paid and how much is left; entries show `3/10`. Deleting a purchase deletes all its installments.
- **Progress** — the *Evolução* tab covers a period you choose (shortcuts for 6 months, 12 months, this year and everything, or any first and last month): average spending, income and balance per month and the highest month; a chart of spending per month stacked by 50/30/20 group with the income line on top; spending per category; and one item over time (pick a description such as *Luz* or *Condomínio* to see its amount month by month).
- **Layout** — dark theme, bottom tab bar on phones (*Mês*, *Lançamentos*, *Fixas*, *Evolução*) and a sidebar on screens 960 px and wider. The floating button is *Lançar* everywhere, and *Nova conta fixa* / *Novo parcelamento* in the *Fixas* tab.

> The user interface is in Brazilian Portuguese.

## Tech stack

| Piece | Used for |
| --- | --- |
| HTML, CSS, vanilla JavaScript (ES modules) | The app itself |
| [supabase-js v2](https://supabase.com/docs/reference/javascript) | Auth and database access |
| [Chart.js v4](https://www.chartjs.org/) | Charts in the *Evolução* tab |

Both libraries are loaded from the jsDelivr CDN.

## Getting started

### 1. Supabase project

The app uses the existing personal Supabase project (shared with the workout tracker). Its tables live in the `financas` schema so they never mix with other apps' tables in `public`.

### 2. Create the schema

Open the **SQL Editor** and run the script below. Since 30 Oct 2026 Supabase no longer grants access to new tables automatically, so every table needs explicit `GRANT`s, Row Level Security and policies. Only the `authenticated` role gets access: the app always requires sign-in.

```sql
create schema if not exists financas;

grant usage on schema financas to authenticated, service_role;

-- Who can read whose data. Maintained from the SQL Editor (the app only reads it)
create table financas.leitores (
  dono_id    uuid not null references auth.users on delete cascade,
  leitor_id  uuid not null references auth.users on delete cascade,
  created_at timestamptz not null default now(),
  primary key (dono_id, leitor_id),
  check (dono_id <> leitor_id)
);

grant select on financas.leitores to authenticated;
grant all    on financas.leitores to service_role;

alter table financas.leitores enable row level security;

create policy "see own sharing" on financas.leitores
  for select to authenticated
  using (auth.uid() = dono_id or auth.uid() = leitor_id);

-- Payment methods: shared list, maintained from the SQL Editor (the app only reads it)
create table financas.meios_pagamento (
  id             bigint generated always as identity primary key,
  nome           text not null unique,
  tipo           text not null check (tipo in ('cartao', 'pix', 'boleto', 'folha', 'beneficio', 'dinheiro', 'outro')),
  -- Credit cards: a purchase on or after the closing day goes to the next bill
  dia_fechamento int check (dia_fechamento between 1 and 31),
  dia_vencimento int check (dia_vencimento between 1 and 31),
  titular        text,
  ativo          boolean not null default true
);

-- Categories: shared list, maintained from the SQL Editor (the app only reads it)
create table financas.categorias (
  id           bigint generated always as identity primary key,
  nome         text not null,
  tipo         text not null default 'despesa' check (tipo in ('despesa', 'receita')),
  -- 50/30/20 group suggested when the category is picked
  grupo_padrao text check (grupo_padrao in ('essencial', 'diversao', 'investimento')),
  ativo        boolean not null default true,
  unique (nome, tipo)
);

-- Fixed bills: one entry per month is created from each of them (pending until paid)
create table financas.contas_fixas (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  tipo              text not null check (tipo in ('despesa', 'receita')),
  descricao         text not null check (length(trim(descricao)) > 0),
  valor_previsto    numeric(12,2) not null check (valor_previsto > 0),
  dia               int not null check (dia between 1 and 31),
  meio_pagamento_id bigint not null references financas.meios_pagamento on delete restrict,
  categoria_id      bigint references financas.categorias on delete restrict,
  grupo             text check (grupo in ('essencial', 'diversao', 'investimento')),
  -- First and last month (first day of the month); no end = fim null
  inicio            date not null check (extract(day from inicio) = 1),
  fim               date check (extract(day from fim) = 1),
  created_at        timestamptz not null default now(),
  check ((tipo = 'despesa') = (grupo is not null)),
  check (fim is null or fim >= inicio)
);

-- Installment purchases: all installments are created as entries when the purchase is saved
create table financas.parcelamentos (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  descricao         text not null check (length(trim(descricao)) > 0),
  valor_parcela     numeric(12,2) not null check (valor_parcela > 0),
  total_parcelas    int not null check (total_parcelas between 2 and 120),
  -- Month installment 1 is paid in; installment n is paid n - 1 months later
  primeiro_mes      date not null check (extract(day from primeiro_mes) = 1),
  data_compra       date not null,
  meio_pagamento_id bigint not null references financas.meios_pagamento on delete restrict,
  categoria_id      bigint references financas.categorias on delete restrict,
  grupo             text not null check (grupo in ('essencial', 'diversao', 'investimento')),
  created_at        timestamptz not null default now()
);

-- Months whose fixed bills were already created (so a deleted entry is not created again)
create table financas.meses_gerados (
  user_id uuid not null default auth.uid() references auth.users on delete cascade,
  mes     date not null check (extract(day from mes) = 1),
  primary key (user_id, mes)
);

create table financas.lancamentos (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  tipo              text not null check (tipo in ('despesa', 'receita')),
  data              date not null default current_date,
  -- Month the entry is paid in (first day of the month). Card purchases go to the bill's due month
  mes               date not null check (extract(day from mes) = 1),
  descricao         text not null check (length(trim(descricao)) > 0),
  -- Negative expense = refund
  valor             numeric(12,2) not null check (valor <> 0),
  grupo             text check (grupo in ('essencial', 'diversao', 'investimento')),
  categoria_id      bigint references financas.categorias on delete restrict,
  meio_pagamento_id bigint references financas.meios_pagamento on delete restrict,
  pago              boolean not null default true,
  observacao        text,
  -- Created from a fixed bill (once per month) or as installment n of a purchase
  conta_fixa_id     bigint references financas.contas_fixas on delete set null,
  parcelamento_id   bigint references financas.parcelamentos on delete cascade,
  parcela           int check (parcela > 0),
  created_at        timestamptz not null default now(),
  -- Expenses always have a 50/30/20 group; income never does
  check ((tipo = 'despesa') = (grupo is not null)),
  constraint lancamentos_conta_fixa_mes unique (conta_fixa_id, mes),
  constraint lancamentos_parcela unique (parcelamento_id, parcela),
  constraint lancamentos_parcela_completa check ((parcelamento_id is null) = (parcela is null))
);

create index lancamentos_mes_idx       on financas.lancamentos (user_id, mes);
create index lancamentos_descricao_idx on financas.lancamentos (user_id, descricao);

grant select on financas.meios_pagamento, financas.categorias to authenticated;
grant all    on financas.meios_pagamento, financas.categorias to service_role;
grant select, insert, update, delete on financas.lancamentos to authenticated;
grant all    on financas.lancamentos to service_role;
grant usage, select on all sequences in schema financas to authenticated, service_role;

alter table financas.meios_pagamento enable row level security;
alter table financas.categorias      enable row level security;
alter table financas.lancamentos     enable row level security;

create policy "read payment methods" on financas.meios_pagamento
  for select to authenticated using (true);

create policy "read categories" on financas.categorias
  for select to authenticated using (true);

-- The owner and the readers listed in leitores can read; only the owner can write
create policy "read own or shared entries" on financas.lancamentos
  for select to authenticated
  using (auth.uid() = user_id
         or exists (select 1 from financas.leitores l
                     where l.dono_id = user_id and l.leitor_id = auth.uid()));

-- An entry can only point to the user's own fixed bill and installment purchase
create policy "insert own entries" on financas.lancamentos
  for insert to authenticated
  with check (
    auth.uid() = user_id
    and (conta_fixa_id is null
         or exists (select 1 from financas.contas_fixas c where c.id = conta_fixa_id and c.user_id = auth.uid()))
    and (parcelamento_id is null
         or exists (select 1 from financas.parcelamentos p where p.id = parcelamento_id and p.user_id = auth.uid()))
  );

create policy "update own entries" on financas.lancamentos
  for update to authenticated
  using (auth.uid() = user_id)
  with check (
    auth.uid() = user_id
    and (conta_fixa_id is null
         or exists (select 1 from financas.contas_fixas c where c.id = conta_fixa_id and c.user_id = auth.uid()))
    and (parcelamento_id is null
         or exists (select 1 from financas.parcelamentos p where p.id = parcelamento_id and p.user_id = auth.uid()))
  );

create policy "delete own entries" on financas.lancamentos
  for delete to authenticated using (auth.uid() = user_id);

grant select, insert, update, delete on financas.contas_fixas, financas.parcelamentos, financas.meses_gerados to authenticated;
grant all on financas.contas_fixas, financas.parcelamentos, financas.meses_gerados to service_role;
grant usage, select on all sequences in schema financas to authenticated, service_role;

alter table financas.contas_fixas  enable row level security;
alter table financas.parcelamentos enable row level security;
alter table financas.meses_gerados enable row level security;

create policy "own fixed bills" on financas.contas_fixas
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "read shared fixed bills" on financas.contas_fixas
  for select to authenticated
  using (exists (select 1 from financas.leitores l where l.dono_id = user_id and l.leitor_id = auth.uid()));

create policy "own installment purchases" on financas.parcelamentos
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "read shared installment purchases" on financas.parcelamentos
  for select to authenticated
  using (exists (select 1 from financas.leitores l where l.dono_id = user_id and l.leitor_id = auth.uid()));

create policy "own generated months" on financas.meses_gerados
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

insert into financas.meios_pagamento (nome, tipo, titular) values
  ('Pix', 'pix', null), ('Boleto', 'boleto', null), ('Desconto em folha', 'folha', null),
  ('CC Inter', 'cartao', null), ('CC Porto', 'cartao', null), ('CC C6 - Vera', 'cartao', 'Vera'),
  ('Caju', 'beneficio', null), ('Dinheiro', 'dinheiro', null);

insert into financas.categorias (nome, tipo, grupo_padrao) values
  ('Alimentação', 'despesa', 'essencial'), ('Restaurantes e delivery', 'despesa', 'diversao'),
  ('Casa', 'despesa', 'essencial'), ('Transporte', 'despesa', 'essencial'),
  ('Saúde', 'despesa', 'essencial'), ('Pet', 'despesa', 'essencial'),
  ('Educação', 'despesa', 'essencial'), ('Roupas', 'despesa', 'essencial'),
  ('Seguros', 'despesa', 'essencial'), ('Impostos e taxas', 'despesa', 'essencial'),
  ('Dívidas', 'despesa', 'essencial'), ('Doações', 'despesa', 'essencial'),
  ('Casamento', 'despesa', 'essencial'), ('Assinaturas', 'despesa', 'diversao'),
  ('Lazer', 'despesa', 'diversao'), ('Viagens', 'despesa', 'diversao'),
  ('Presentes', 'despesa', 'diversao'), ('Investimentos', 'despesa', 'investimento'),
  ('Outros', 'despesa', null),
  ('Salário', 'receita', null), ('Renda extra', 'receita', null), ('Outros', 'receita', null);
```

| Table | Purpose |
| --- | --- |
| `leitores` | Read-only sharing: `leitor_id` can read the data of `dono_id` (owner). One row per owner and reader |
| `meios_pagamento` | Payment methods (`nome`, `tipo`). Credit cards (`tipo = 'cartao'`) also have `dia_fechamento` (closing day) and `dia_vencimento` (due day), used to put each purchase in the month its bill is paid; `titular` = card holder when it is someone else's card. Shared list, edited only in the SQL Editor |
| `categorias` | Categories (`nome`, `tipo` = expense or income, `grupo_padrao` = the 50/30/20 group suggested when picked). Shared list, edited only in the SQL Editor; `ativo = false` hides one |
| `contas_fixas` | Fixed bills and recurring income (`descricao`, `valor_previsto`, `dia` of the month, payment method, category, group, `inicio` / `fim` = first and last month). Every month the app creates one pending entry from each of them |
| `parcelamentos` | Installment purchases (`valor_parcela`, `total_parcelas`, `primeiro_mes` = month installment 1 is paid in, `data_compra`). All installments are created as entries when the purchase is saved |
| `meses_gerados` | Months whose fixed-bill entries were already created, so an entry deleted on purpose is not created again |
| `lancamentos` | Entries (`tipo` = `despesa` or `receita`, `data`, `mes` = first day of the month it is paid in, `descricao`, `valor`, `grupo` = `essencial` / `diversao` / `investimento` for expenses, `categoria_id`, `meio_pagamento_id`, `pago` = already paid, `observacao`, `conta_fixa_id` or `parcelamento_id` + `parcela` when it came from a fixed bill or an installment purchase). A negative expense is a refund |

Set the closing and due day of each credit card (the values below are examples):

```sql
update financas.meios_pagamento set dia_fechamento = 5,  dia_vencimento = 15 where nome = 'CC C6 - Vera';
update financas.meios_pagamento set dia_fechamento = 3,  dia_vencimento = 10 where nome = 'CC Inter';
update financas.meios_pagamento set dia_fechamento = 28, dia_vencimento = 5  where nome = 'CC Porto';
```

Until a card has both days, its purchases count in the month of the purchase date.

To give someone read access, create their account first (step 4) and then run, with the real addresses:

```sql
insert into financas.leitores (dono_id, leitor_id)
select dono.id, leitor.id
  from auth.users dono, auth.users leitor
 where dono.email = 'owner@example.com'
   and leitor.email = 'reader@example.com';
```

#### Upgrading an existing database

Run the upgrades your database still needs, in this order, in the SQL Editor **before** deploying the new `index.html`.

**Entries, payment methods and categories.** Creates the `meios_pagamento`, `categorias` and `lancamentos` tables, fills in the two lists and gives readers in `leitores` read access to the owner's entries. Run it on a database created with the first version of this README (only `leitores`):

```sql
begin;

-- Payment methods: shared list, maintained from the SQL Editor (the app only reads it)
create table financas.meios_pagamento (
  id             bigint generated always as identity primary key,
  nome           text not null unique,
  tipo           text not null check (tipo in ('cartao', 'pix', 'boleto', 'folha', 'beneficio', 'dinheiro', 'outro')),
  -- Credit cards: a purchase on or after the closing day goes to the next bill
  dia_fechamento int check (dia_fechamento between 1 and 31),
  dia_vencimento int check (dia_vencimento between 1 and 31),
  titular        text,
  ativo          boolean not null default true
);

-- Categories: shared list, maintained from the SQL Editor (the app only reads it)
create table financas.categorias (
  id           bigint generated always as identity primary key,
  nome         text not null,
  tipo         text not null default 'despesa' check (tipo in ('despesa', 'receita')),
  -- 50/30/20 group suggested when the category is picked
  grupo_padrao text check (grupo_padrao in ('essencial', 'diversao', 'investimento')),
  ativo        boolean not null default true,
  unique (nome, tipo)
);

create table financas.lancamentos (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  tipo              text not null check (tipo in ('despesa', 'receita')),
  data              date not null default current_date,
  -- Month the entry is paid in (first day of the month). Card purchases go to the bill's due month
  mes               date not null check (extract(day from mes) = 1),
  descricao         text not null check (length(trim(descricao)) > 0),
  -- Negative expense = refund
  valor             numeric(12,2) not null check (valor <> 0),
  grupo             text check (grupo in ('essencial', 'diversao', 'investimento')),
  categoria_id      bigint references financas.categorias on delete restrict,
  meio_pagamento_id bigint references financas.meios_pagamento on delete restrict,
  pago              boolean not null default true,
  observacao        text,
  created_at        timestamptz not null default now(),
  -- Expenses always have a 50/30/20 group; income never does
  check ((tipo = 'despesa') = (grupo is not null))
);

create index lancamentos_mes_idx       on financas.lancamentos (user_id, mes);
create index lancamentos_descricao_idx on financas.lancamentos (user_id, descricao);

grant select on financas.meios_pagamento, financas.categorias to authenticated;
grant all    on financas.meios_pagamento, financas.categorias to service_role;
grant select, insert, update, delete on financas.lancamentos to authenticated;
grant all    on financas.lancamentos to service_role;
grant usage, select on all sequences in schema financas to authenticated, service_role;

alter table financas.meios_pagamento enable row level security;
alter table financas.categorias      enable row level security;
alter table financas.lancamentos     enable row level security;

create policy "read payment methods" on financas.meios_pagamento
  for select to authenticated using (true);

create policy "read categories" on financas.categorias
  for select to authenticated using (true);

-- The owner and the readers listed in leitores can read; only the owner can write
create policy "read own or shared entries" on financas.lancamentos
  for select to authenticated
  using (auth.uid() = user_id
         or exists (select 1 from financas.leitores l
                     where l.dono_id = user_id and l.leitor_id = auth.uid()));

create policy "insert own entries" on financas.lancamentos
  for insert to authenticated with check (auth.uid() = user_id);

create policy "update own entries" on financas.lancamentos
  for update to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "delete own entries" on financas.lancamentos
  for delete to authenticated using (auth.uid() = user_id);

insert into financas.meios_pagamento (nome, tipo, titular) values
  ('Pix', 'pix', null), ('Boleto', 'boleto', null), ('Desconto em folha', 'folha', null),
  ('CC Inter', 'cartao', null), ('CC Porto', 'cartao', null), ('CC C6 - Vera', 'cartao', 'Vera'),
  ('Caju', 'beneficio', null), ('Dinheiro', 'dinheiro', null);

insert into financas.categorias (nome, tipo, grupo_padrao) values
  ('Alimentação', 'despesa', 'essencial'), ('Restaurantes e delivery', 'despesa', 'diversao'),
  ('Casa', 'despesa', 'essencial'), ('Transporte', 'despesa', 'essencial'),
  ('Saúde', 'despesa', 'essencial'), ('Pet', 'despesa', 'essencial'),
  ('Educação', 'despesa', 'essencial'), ('Roupas', 'despesa', 'essencial'),
  ('Seguros', 'despesa', 'essencial'), ('Impostos e taxas', 'despesa', 'essencial'),
  ('Dívidas', 'despesa', 'essencial'), ('Doações', 'despesa', 'essencial'),
  ('Casamento', 'despesa', 'essencial'), ('Assinaturas', 'despesa', 'diversao'),
  ('Lazer', 'despesa', 'diversao'), ('Viagens', 'despesa', 'diversao'),
  ('Presentes', 'despesa', 'diversao'), ('Investimentos', 'despesa', 'investimento'),
  ('Outros', 'despesa', null),
  ('Salário', 'receita', null), ('Renda extra', 'receita', null), ('Outros', 'receita', null);

commit;
```

**Fixed bills and installment purchases.** Creates `contas_fixas`, `parcelamentos` and `meses_gerados`, links entries to them and makes sure an entry can only point to the user's own fixed bill and purchase. Existing entries are not changed. Run it after the *Entries, payment methods and categories* upgrade:

```sql
begin;

-- Fixed bills: one entry per month is created from each of them (pending until paid)
create table financas.contas_fixas (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  tipo              text not null check (tipo in ('despesa', 'receita')),
  descricao         text not null check (length(trim(descricao)) > 0),
  valor_previsto    numeric(12,2) not null check (valor_previsto > 0),
  dia               int not null check (dia between 1 and 31),
  meio_pagamento_id bigint not null references financas.meios_pagamento on delete restrict,
  categoria_id      bigint references financas.categorias on delete restrict,
  grupo             text check (grupo in ('essencial', 'diversao', 'investimento')),
  -- First and last month (first day of the month); no end = fim null
  inicio            date not null check (extract(day from inicio) = 1),
  fim               date check (extract(day from fim) = 1),
  created_at        timestamptz not null default now(),
  check ((tipo = 'despesa') = (grupo is not null)),
  check (fim is null or fim >= inicio)
);

-- Installment purchases: all installments are created as entries when the purchase is saved
create table financas.parcelamentos (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  descricao         text not null check (length(trim(descricao)) > 0),
  valor_parcela     numeric(12,2) not null check (valor_parcela > 0),
  total_parcelas    int not null check (total_parcelas between 2 and 120),
  -- Month installment 1 is paid in; installment n is paid n - 1 months later
  primeiro_mes      date not null check (extract(day from primeiro_mes) = 1),
  data_compra       date not null,
  meio_pagamento_id bigint not null references financas.meios_pagamento on delete restrict,
  categoria_id      bigint references financas.categorias on delete restrict,
  grupo             text not null check (grupo in ('essencial', 'diversao', 'investimento')),
  created_at        timestamptz not null default now()
);

-- Months whose fixed bills were already created (so a deleted entry is not created again)
create table financas.meses_gerados (
  user_id uuid not null default auth.uid() references auth.users on delete cascade,
  mes     date not null check (extract(day from mes) = 1),
  primary key (user_id, mes)
);

alter table financas.lancamentos
  add column conta_fixa_id   bigint references financas.contas_fixas on delete set null,
  add column parcelamento_id bigint references financas.parcelamentos on delete cascade,
  add column parcela         int check (parcela > 0),
  add constraint lancamentos_conta_fixa_mes unique (conta_fixa_id, mes),
  add constraint lancamentos_parcela unique (parcelamento_id, parcela),
  add constraint lancamentos_parcela_completa check ((parcelamento_id is null) = (parcela is null));

grant select, insert, update, delete on financas.contas_fixas, financas.parcelamentos, financas.meses_gerados to authenticated;
grant all on financas.contas_fixas, financas.parcelamentos, financas.meses_gerados to service_role;
grant usage, select on all sequences in schema financas to authenticated, service_role;

alter table financas.contas_fixas  enable row level security;
alter table financas.parcelamentos enable row level security;
alter table financas.meses_gerados enable row level security;

create policy "own fixed bills" on financas.contas_fixas
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "read shared fixed bills" on financas.contas_fixas
  for select to authenticated
  using (exists (select 1 from financas.leitores l where l.dono_id = user_id and l.leitor_id = auth.uid()));

create policy "own installment purchases" on financas.parcelamentos
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "read shared installment purchases" on financas.parcelamentos
  for select to authenticated
  using (exists (select 1 from financas.leitores l where l.dono_id = user_id and l.leitor_id = auth.uid()));

create policy "own generated months" on financas.meses_gerados
  for all to authenticated using (auth.uid() = user_id) with check (auth.uid() = user_id);

drop policy "insert own entries" on financas.lancamentos;
drop policy "update own entries" on financas.lancamentos;

-- An entry can only point to the user's own fixed bill and installment purchase
create policy "insert own entries" on financas.lancamentos
  for insert to authenticated
  with check (
    auth.uid() = user_id
    and (conta_fixa_id is null
         or exists (select 1 from financas.contas_fixas c where c.id = conta_fixa_id and c.user_id = auth.uid()))
    and (parcelamento_id is null
         or exists (select 1 from financas.parcelamentos p where p.id = parcelamento_id and p.user_id = auth.uid()))
  );

create policy "update own entries" on financas.lancamentos
  for update to authenticated
  using (auth.uid() = user_id)
  with check (
    auth.uid() = user_id
    and (conta_fixa_id is null
         or exists (select 1 from financas.contas_fixas c where c.id = conta_fixa_id and c.user_id = auth.uid()))
    and (parcelamento_id is null
         or exists (select 1 from financas.parcelamentos p where p.id = parcelamento_id and p.user_id = auth.uid()))
  );

commit;
```

### 3. Expose the schema to the Data API

In **Project Settings → API** (Data API settings), add `financas` to **Exposed schemas**. **Keep `public` in the list** — the workout tracker uses it.

### 4. Auth

- Create the accounts in **Authentication → Users → Add user** (or reuse the accounts already created in the workout tracker: it is the same project).
- In **Authentication → URL Configuration → Redirect URLs**, add `https://account.julianodrebel.com.br` and the Worker preview URL, so Google sign-in returns to this app.
- Once every account exists, turn off sign-ups for new users in the Auth settings: the database is personal.

### 5. Configure the app

The project URL, the **publishable** key and the schema name are at the top of the `<script type="module">` block in `index.html`:

```js
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-publishable-key';
const SCHEMA = 'financas';
```

The publishable key is designed to be exposed in client-side code; the data is protected by the grants and Row Level Security policies above. Never put the `service_role` / secret key in this file.

### 6. Run it

Because the app uses ES modules, serve it over HTTP instead of opening the file directly:

```bash
npx serve .
# or
python -m http.server 8000
```

### 7. Import the old spreadsheet (once)

The monthly Google Sheets spreadsheet this app replaces can be imported with `tools/importar_planilha.py`. Export it as `.xlsx` and run:

```bash
pip install openpyxl
python tools/importar_planilha.py Planilha_de_gastos.xlsx            # owner = dono_id in financas.leitores
python tools/importar_planilha.py Planilha_de_gastos.xlsx --email you@example.com
```

It writes `tools/saida/importacao.sql`, to be run once in the SQL Editor (after all the upgrades above). That SQL:

- imports every tab as entries in the tab's month (each row's own date is kept), paid unless *Pago* is unticked, plus the salary from the *RECEBIDO* block as income;
- cleans the data: payment method spellings (`PIX`, `PX` → *Pix*, `CC C6` → *CC C6 - Vera*; unknown ones are left empty), the 50/30/20 group from *Categoria* (rows without one get the group most used for the same description), the category from *Sub Categoria* or, without one, from words in the description (`Netflix` → *Assinaturas*, `Mercado` → *Alimentação*…); rows without an amount and the total row are skipped;
- creates the fixed bills (rows with *Custo Fixo = Sim* and no installment) and the installment purchases still running (`GoPro - 2/10`) from the last tab, starting the month after it; fixed bills or purchases with the same description that already exist are skipped, and months the app already generated get the new fixed bills' entries;
- checks that each month's total of expenses equals the sum of the tab's *Valor* column, and rolls everything back if one does not match;
- refuses to run again once there are entries before that month.

The spreadsheet and the generated SQL contain personal data and are git-ignored.

## Deploying

The site is a Cloudflare Worker serving static assets (`wrangler.jsonc`), connected to this GitHub repository: every push deploys (`master` to production at `account.julianodrebel.com.br`, other branches to preview URLs). `.assetsignore` keeps everything except `index.html` off the site.

## Branching model

- `master` — stable, production-ready code.
- `develop` — integration branch for ongoing work. Create feature branches from `develop` and merge them back via pull request.
