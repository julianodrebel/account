# 💰 Minhas Finanças

A lightweight, mobile-first personal finance tracker that runs entirely in the browser. It replaces a monthly spreadsheet: log expenses in a few taps, see what is still to be paid this month, and track spending against the 50/30/20 rule. Data lives in [Supabase](https://supabase.com), in the same project as the [workout tracker](https://github.com/julianodrebel/gym), under its own `financas` schema.

The whole app is a single `index.html` file: no build step, no bundler, no server code.

## Features

- **Accounts** — sign in with email and password or with Google (Supabase Auth). There is no sign-up screen: accounts are created in the Supabase dashboard.
- **Read-only sharing** — a second person can be given read access to the owner's data (`financas.leitores`). They see everything, but the app hides every action that changes data and the database refuses writes.
- **Layout** — dark theme, bottom tab bar on phones (*Mês*, *Lançamentos*, *Fixas*, *Evolução*) and a sidebar on screens 960 px and wider. The *Mês* tab has a month picker; the other tabs are placeholders for now.

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
```

| Table | Purpose |
| --- | --- |
| `leitores` | Read-only sharing: `leitor_id` can read the data of `dono_id` (owner). One row per owner and reader |

To give someone read access, create their account first (step 4) and then run, with the real addresses:

```sql
insert into financas.leitores (dono_id, leitor_id)
select dono.id, leitor.id
  from auth.users dono, auth.users leitor
 where dono.email = 'owner@example.com'
   and leitor.email = 'reader@example.com';
```

#### Upgrading an existing database

No upgrades yet.

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
