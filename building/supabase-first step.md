Sí. Este es el **paso a paso base** para armar byDebut en Supabase antes del frontend.

Supabase será: **Auth + PostgreSQL + Storage + seguridad + funciones IA**. RLS debe estar activo para controlar qué datos puede ver o modificar cada usuario. ([Supabase][1])

---

# Paso 1 — Crear proyecto

En Supabase:

1. New Project.
2. Nombre: `bydebut`.
3. Region: cercana a tus usuarios iniciales.
4. Guarda:

   * Project URL
   * Anon/Public Key
   * Service Role Key, solo backend
5. Ve a **SQL Editor**.

---

# Paso 2 — Crear extensiones

Ejecuta:

```sql
create extension if not exists "pgcrypto";
create extension if not exists "uuid-ossp";
```

---

# Paso 3 — Crear tablas principales

Ejecuta todo esto en SQL Editor:

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text,
  email text,
  role text default 'reader',
  avatar_url text,
  created_at timestamptz default now()
);

create table public.books (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  subtitle text,
  author_name text,
  language text default 'en',
  description text,
  cover_url text,
  public_domain boolean default false,
  owner_id uuid references public.profiles(id),
  status text default 'active',
  created_at timestamptz default now()
);

create table public.chapters (
  id uuid primary key default gen_random_uuid(),
  book_id uuid references public.books(id) on delete cascade,
  chapter_number int,
  title text,
  content text,
  created_at timestamptz default now()
);

create table public.transformation_modes (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  is_mvp boolean default true,
  is_beta boolean default false,
  created_at timestamptz default now()
);

create table public.versions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  book_id uuid references public.books(id) on delete cascade,
  chapter_id uuid references public.chapters(id),
  parent_version_id uuid references public.versions(id),
  mode_id uuid references public.transformation_modes(id),
  title text,
  prompt_used text,
  generated_content text,
  visibility text default 'private',
  status text default 'draft',
  is_beta_feature boolean default false,
  created_at timestamptz default now()
);

create table public.version_variables (
  id uuid primary key default gen_random_uuid(),
  version_id uuid references public.versions(id) on delete cascade,
  variable_name text,
  variable_value text,
  created_at timestamptz default now()
);

create table public.ratings (
  id uuid primary key default gen_random_uuid(),
  version_id uuid references public.versions(id) on delete cascade,
  user_id uuid references public.profiles(id) on delete cascade,
  rating int check (rating between 1 and 5),
  comment text,
  created_at timestamptz default now(),
  unique(version_id, user_id)
);

create table public.subscriptions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  plan_name text default 'free',
  monthly_version_limit int default 5,
  versions_used int default 0,
  status text default 'active',
  created_at timestamptz default now()
);

create table public.beta_features (
  id uuid primary key default gen_random_uuid(),
  feature_name text not null,
  description text,
  target_user text,
  revenue_potential text,
  status text default 'beta',
  created_at timestamptz default now()
);
```

---

# Paso 4 — Activar seguridad RLS

```sql
alter table public.profiles enable row level security;
alter table public.books enable row level security;
alter table public.chapters enable row level security;
alter table public.transformation_modes enable row level security;
alter table public.versions enable row level security;
alter table public.version_variables enable row level security;
alter table public.ratings enable row level security;
alter table public.subscriptions enable row level security;
alter table public.beta_features enable row level security;
```

---

# Paso 5 — Crear políticas de acceso

```sql
create policy "Users can view own profile"
on public.profiles
for select
using (auth.uid() = id);

create policy "Users can update own profile"
on public.profiles
for update
using (auth.uid() = id);

create policy "Anyone can read active books"
on public.books
for select
using (status = 'active');

create policy "Anyone can read chapters"
on public.chapters
for select
using (true);

create policy "Anyone can read transformation modes"
on public.transformation_modes
for select
using (true);

create policy "Users can read own versions"
on public.versions
for select
using (auth.uid() = user_id or visibility = 'public');

create policy "Users can create own versions"
on public.versions
for insert
with check (auth.uid() = user_id);

create policy "Users can update own versions"
on public.versions
for update
using (auth.uid() = user_id);

create policy "Users can delete own versions"
on public.versions
for delete
using (auth.uid() = user_id);

create policy "Users can read variables of own versions"
on public.version_variables
for select
using (
  exists (
    select 1 from public.versions
    where versions.id = version_variables.version_id
    and versions.user_id = auth.uid()
  )
);

create policy "Users can insert variables for own versions"
on public.version_variables
for insert
with check (
  exists (
    select 1 from public.versions
    where versions.id = version_variables.version_id
    and versions.user_id = auth.uid()
  )
);

create policy "Anyone can read ratings"
on public.ratings
for select
using (true);

create policy "Users can insert own ratings"
on public.ratings
for insert
with check (auth.uid() = user_id);

create policy "Users can read own subscription"
on public.subscriptions
for select
using (auth.uid() = user_id);

create policy "Anyone can read beta features"
on public.beta_features
for select
using (true);
```

---

# Paso 6 — Insertar datos MVP y BETA

```sql
insert into public.transformation_modes (name, description, is_mvp, is_beta)
values
('Literary Variation', 'Create a new literary version from the selected text.', true, false),
('Cinematic Variation', 'Transform the text into a film or series-oriented version.', true, false),
('Systemic Analysis', 'Analyze the story through systems, incentives, conflicts, and consequences.', true, false),
('Full Novel Expansion', 'Expand a short idea or chapter into a longer novel structure.', true, true),
('Children Illustrated Version', 'Create a child-friendly illustrated adaptation.', false, true),
('Publisher Licensing Package', 'Prepare the version for commercial review and rights evaluation.', false, true),
('Audio / Video Adaptation', 'Prepare the story for audio, video, or streaming formats.', false, true);

insert into public.beta_features (feature_name, description, target_user, revenue_potential)
values
('Publisher Marketplace', 'Publishers can discover high-performing versions.', 'Publisher', 'Licensing, subscription, commission'),
('Author Revenue Dashboard', 'Authors can track versions, engagement, and monetization.', 'Author', 'Author Pro subscription'),
('Producer Studio Access', 'Film and streaming producers can evaluate stories for adaptation.', 'Producer', 'Premium B2B access'),
('Children Illustrated Adaptation', 'Create age-specific illustrated versions.', 'Reader / Parent / Publisher', 'Premium generation package'),
('Version Licensing', 'Allow commercial licensing of selected versions.', 'Author / Publisher / Producer', 'Transaction fees'),
('Multi-language Publishing', 'Generate multilingual versions for global publishing.', 'Publisher', 'Enterprise subscription');
```

---

# Paso 7 — Crear libros demo

```sql
insert into public.books (title, subtitle, author_name, language, description, public_domain, status)
values
('The Only Option', 'AI, Mars, and the future of civilization', 'Iván Padrón', 'en', 'A speculative story about humanity, artificial intelligence, and survival.', false, 'active'),
('Lethal Eternity', 'The first limit of immortality', 'Iván Padrón', 'en', 'A literary exploration of immortality and the collapse of human behavior without mortality.', false, 'active'),
('The Devil’s Throats', 'The invisible system that sustains and threatens the global economy', 'Iván Padrón', 'en', 'A geopolitical story about chokepoints, trade, war, and fragile global systems.', false, 'active');
```

Después agrega manualmente 2 o 3 capítulos demo por libro desde **Table Editor > chapters**.

---

# Paso 8 — Crear Storage

En Supabase > Storage crea buckets:

```text
book-covers
book-files
user-uploads
generated-assets
```

Recomendación:

```text
book-covers = public
book-files = private
user-uploads = private
generated-assets = private
```

Supabase Storage usa políticas de acceso conectadas con RLS; por defecto no permite uploads sin políticas configuradas. ([Supabase][2])

---

# Paso 9 — Separar MVP vs BETA en la base de datos

La clave está en estos campos:

```text
is_mvp
is_beta
status
visibility
```

En el frontend:

```text
MVP = is_mvp = true
BETA = is_beta = true
```

Así el jurado verá:

```text
Disponible ahora:
- Library
- Version Generator
- Saved Versions
- Version Tree
- Ranking

BETA:
- Publisher Marketplace
- Studio Access
- Licensing
- Audio/Video
- Revenue Dashboard
```

---

# Paso 10 — Preparar Edge Function para IA

Más adelante crearás una función:

```text
generate-version
```

Su trabajo será:

```text
recibir usuario + libro + capítulo + modo + variables
leer contenido
llamar Gemini API
guardar resultado en versions
devolver resultado al frontend
```

Esto es importante porque **la API key de Gemini no debe ir en el frontend**. Supabase permite crear funciones de base de datos desde SQL Editor y también Edge Functions para lógica server-side. ([Supabase][3])

---

# Paso 11 — Arquitectura final inicial

```text
Frontend Claude / Lovable
        |
        v
Supabase Auth
        |
        v
Supabase PostgreSQL
        |
        v
Supabase Storage
        |
        v
Supabase Edge Function
        |
        v
Gemini API
```

Después:

```text
Supabase
   |
   v
n8n = automatización operativa

OpenClaw = inteligencia comercial futura
```

---

# Paso 12 — Qué hacer inmediatamente

Tu primer trabajo en Supabase es este orden:

```text
1. Crear proyecto
2. Ejecutar SQL de tablas
3. Activar RLS
4. Crear policies
5. Insertar transformation_modes
6. Insertar beta_features
7. Insertar libros demo
8. Crear buckets de Storage
9. Probar login
10. Luego pedirle a Claude el frontend conectado a Supabase
```

Mi recomendación final: **no construyas primero un frontend bonito**. Construye primero esta base. Después Claude o Lovable tendrán una estructura real sobre la cual trabajar.

[1]: https://supabase.com/docs/guides/database/postgres/row-level-security?utm_source=chatgpt.com "Row Level Security | Supabase Docs"
[2]: https://supabase.com/docs/guides/storage/security/access-control?utm_source=chatgpt.com "Storage Access Control | Supabase Docs"
[3]: https://supabase.com/docs/guides/database/functions?utm_source=chatgpt.com "Database Functions | Supabase Docs"
