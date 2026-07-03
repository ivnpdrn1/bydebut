
Perfecto. Empezamos por el **backend en Supabase**.

# byDebut Backend Build — Paso 1

## Objetivo de esta primera etapa

Crear la base estructural:

```text
Supabase
↓
PostgreSQL
↓
pgvector
↓
Auth
↓
Books
↓
Chapters
↓
Versions
↓
Story Graph
↓
AI Jobs
```

---

## Paso 1.1 — En Supabase

Ve a:

```text
Supabase Dashboard
↓
SQL Editor
↓
New Query
```

Y ejecuta este primer bloque.

---

```sql
-- ============================================
-- byDebut Backend Foundation
-- Step 1: Extensions
-- ============================================

create extension if not exists "uuid-ossp";
create extension if not exists "pgcrypto";
create extension if not exists "vector";
```

---

## Paso 1.2 — Crear enums principales

Ejecuta después este bloque:

```sql
-- ============================================
-- byDebut Backend Foundation
-- Step 2: Enums
-- ============================================

create type user_role as enum (
  'reader',
  'author',
  'editor',
  'publisher',
  'producer',
  'admin'
);

create type book_status as enum (
  'draft',
  'processing',
  'ready',
  'archived',
  'deleted'
);

create type version_status as enum (
  'draft',
  'generating',
  'ready',
  'failed',
  'archived'
);

create type ai_job_status as enum (
  'pending',
  'running',
  'completed',
  'failed'
);

create type ai_job_type as enum (
  'extract_text',
  'analyze_story',
  'generate_version',
  'create_embedding',
  'compare_versions'
);
```

---

## Paso 1.3 — Crear tabla de perfiles

```sql
-- ============================================
-- Step 3: Profiles
-- ============================================

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text unique not null,
  full_name text,
  avatar_url text,
  role user_role not null default 'reader',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

---

## Paso 1.4 — Crear tabla de libros

```sql
-- ============================================
-- Step 4: Books
-- ============================================

create table public.books (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references public.profiles(id) on delete cascade,

  title text not null,
  subtitle text,
  author_name text,
  description text,

  original_file_url text,
  cover_url text,

  status book_status not null default 'draft',

  language text default 'en',
  genre text,
  word_count integer default 0,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

---

## Paso 1.5 — Crear capítulos

```sql
-- ============================================
-- Step 5: Chapters
-- ============================================

create table public.chapters (
  id uuid primary key default gen_random_uuid(),
  book_id uuid not null references public.books(id) on delete cascade,

  chapter_number integer not null,
  title text,
  content text not null,

  word_count integer default 0,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique(book_id, chapter_number)
);
```

---

## Paso 1.6 — Crear versiones narrativas

Esta tabla es el corazón inicial de byDebut.

```sql
-- ============================================
-- Step 6: Story Versions
-- ============================================

create table public.story_versions (
  id uuid primary key default gen_random_uuid(),

  book_id uuid not null references public.books(id) on delete cascade,
  parent_version_id uuid references public.story_versions(id) on delete set null,

  created_by uuid not null references public.profiles(id) on delete cascade,

  title text not null,
  description text,
  content text not null,

  branch_name text,
  generation_prompt text,

  status version_status not null default 'draft',

  depth integer not null default 0,
  is_original boolean not null default false,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

---

## Paso 1.7 — Crear edges del Story Graph

```sql
-- ============================================
-- Step 7: Story Graph Edges
-- ============================================

create table public.story_graph_edges (
  id uuid primary key default gen_random_uuid(),

  book_id uuid not null references public.books(id) on delete cascade,

  source_version_id uuid not null references public.story_versions(id) on delete cascade,
  target_version_id uuid not null references public.story_versions(id) on delete cascade,

  relationship_type text not null default 'branch',

  created_at timestamptz not null default now(),

  unique(source_version_id, target_version_id)
);
```

---

## Paso 1.8 — Crear AI Jobs

```sql
-- ============================================
-- Step 8: AI Jobs
-- ============================================

create table public.ai_jobs (
  id uuid primary key default gen_random_uuid(),

  user_id uuid not null references public.profiles(id) on delete cascade,
  book_id uuid references public.books(id) on delete cascade,
  version_id uuid references public.story_versions(id) on delete cascade,

  job_type ai_job_type not null,
  status ai_job_status not null default 'pending',

  input jsonb,
  output jsonb,
  error_message text,

  started_at timestamptz,
  completed_at timestamptz,

  created_at timestamptz not null default now()
);
```

---

# Qué acabamos de construir

Ya tenemos la columna vertebral del backend:

```text
User
↓
Book
↓
Chapters
↓
Story Versions
↓
Story Graph
↓
AI Jobs
```

Este es el primer contrato real para Claude Code.

El siguiente paso será crear:

```text
embeddings
story_analysis
characters
events
locations
RLS security policies
storage buckets
```

Continúa con este bloque primero en Supabase.
