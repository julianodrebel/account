# 💰 Minhas Finanças

A lightweight, mobile-first personal finance tracker that runs entirely in the browser. It replaces a monthly spreadsheet: log expenses in a few taps, see what is still to be paid this month, and track spending against the 50/30/20 rule. Data lives in [Supabase](https://supabase.com), in the same project as the [workout tracker](https://github.com/julianodrebel/gym), under its own `financas` schema.

The whole app is a single `index.html` file: no build step, no bundler, no server code.

## Features

- **Accounts** — sign in with email and password or with Google (Supabase Auth). There is no sign-up screen: accounts are created in the Supabase dashboard.
- **Read-only sharing** — a second person can be given read access to the owner's data (`financas.leitores`). They see everything, but the app hides every action that changes data and the database refuses writes.
- **Quick entry** — the *Lançar* button opens a sheet to log an expense or income: amount (typed as `59,90` or `1.234,56`), description, date, payment method, 50/30/20 group (*Essencial*, *Diversão*, *Investimento*), category, whether it is already paid, refund (stored as a negative expense) and a note. Picking a description used before fills in its type, group, category, payment method and amount from the last time. Picking a category suggests its group.
- **Month it is paid in** — every entry belongs to the month it is paid in. For credit cards with closing and due days set, the app puts the purchase in its bill's due month (a purchase on 20 Nov with closing on the 10th and due on the 15th goes to December) and leaves it as *a pagar* (to pay); the month can also be picked by hand.
- **Entries** — the *Lançamentos* tab lists the selected month's entries grouped by day, with income, expenses and balance for what is shown. Search by description, category, payment method or note, and filter by group, income or entries still to pay. Tap an entry to edit or delete it.
- **Layout** — dark theme, bottom tab bar on phones (*Mês*, *Lançamentos*, *Fixas*, *Evolução*) and a sidebar on screens 960 px and wider. *Mês*, *Fixas* and *Evolução* are placeholders for now.

> The user interface is in Brazilian Portuguese.

## Tech stack

| Piece | Used for |
| --- | --- |
| HTML, CSS, vanilla JavaScript (ES modules) | The app itself |
| [supabase-js v2](https://supabase.com/docs/reference/javascript) | Auth and database access |
| [Chart.js v4](https://www.chartjs.org/) | Charts (upcoming) |

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
```

| Table | Purpose |
| --- | --- |
| `leitores` | Read-only sharing: `leitor_id` can read the data of `dono_id` (owner). One row per owner and reader |
| `meios_pagamento` | Payment methods (`nome`, `tipo`). Credit cards (`tipo = 'cartao'`) also have `dia_fechamento` (closing day) and `dia_vencimento` (due day), used to put each purchase in the month its bill is paid; `titular` = card holder when it is someone else's card. Shared list, edited only in the SQL Editor |
| `categorias` | Categories (`nome`, `tipo` = expense or income, `grupo_padrao` = the 50/30/20 group suggested when picked). Shared list, edited only in the SQL Editor; `ativo = false` hides one |
| `lancamentos` | Entries (`tipo` = `despesa` or `receita`, `data`, `mes` = first day of the month it is paid in, `descricao`, `valor`, `grupo` = `essencial` / `diversao` / `investimento` for expenses, `categoria_id`, `meio_pagamento_id`, `pago` = already paid, `observacao`). A negative expense is a refund |

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

## Deploying

The site is a Cloudflare Worker serving static assets (`wrangler.jsonc`), connected to this GitHub repository: every push deploys (`master` to production at `account.julianodrebel.com.br`, other branches to preview URLs). `.assetsignore` keeps everything except `index.html` off the site.

## Branching model

- `master` — stable, production-ready code.
- `develop` — integration branch for ongoing work. Create feature branches from `develop` and merge them back via pull request.
