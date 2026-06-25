ARQUITECTURA BACKEND BYDEBUT — SUPABASE + GEMINI XPRIZE

1. OBJETIVO GENERAL

Crear en Supabase toda la estructura backend de byDebut para que Lovable pueda conectarse al frontend y mostrar al jurado:

MVP disponible:
- Registro/login de usuarios.
- Biblioteca de libros.
- Capítulos.
- Creación de versiones.
- Guardado de versiones.
- Ranking básico.
- Historial del usuario.
- Generación IA usando Gemini.
- Embeddings con Gemini.
- Búsqueda semántica con Supabase pgvector.

BETA visible:
- Árbol de versiones.
- Versiones de versiones.
- Review general y especializado.
- Línea infantil.
- Modo productor/director.
- Monetización por suscripción.
- Marketplace editorial.
- Automatizaciones futuras con n8n/OpenClaw.

2. TECNOLOGÍAS PRINCIPALES

Frontend:
- Lovable.

Backend:
- Supabase.

Base de datos:
- PostgreSQL dentro de Supabase.

Vector database:
- pgvector dentro de Supabase.

IA:
- Gemini API.
- Para embeddings: gemini-embedding-001.
- Para generación: Gemini 1.5 / Gemini 2.x según disponibilidad del hackathon.

NO usar OpenAI en la versión del Hackathon, para mantener alineación con Gemini XPRIZE.

3. EXTENSIONES EN SUPABASE

Ejecutar en SQL Editor:

create extension if not exists vector with schema extensions;
create extension if not exists pgcrypto;

4. TABLAS PRINCIPALES MVP

4.1 profiles

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text,
  role text default 'reader',
  avatar_url text,
  country text,
  language text default 'es',
  created_at timestamptz default now()
);

Roles posibles:
- reader
- creator
- author
- editor
- publisher
- producer
- admin

4.2 books

create table public.books (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  subtitle text,
  author_name text,
  language text default 'es',
  description text,
  genre text,
  cover_url text,
  visibility text default 'public',
  status text default 'active',
  created_by uuid references public.profiles(id),
  created_at timestamptz default now()
);

4.3 chapters

create table public.chapters (
  id uuid primary key default gen_random_uuid(),
  book_id uuid references public.books(id) on delete cascade,
  chapter_number int,
  title text,
  content text not null,
  created_at timestamptz default now()
);

4.4 chapter_embeddings

IMPORTANTE:
Si usamos gemini-embedding-001 en dimensión completa, usar vector(3072).
Si decidimos reducir costo/espacio, usar vector(1536) o vector(768), pero debe ser consistente en toda la base.

create table public.chapter_embeddings (
  id uuid primary key default gen_random_uuid(),
  chapter_id uuid references public.chapters(id) on delete cascade,
  book_id uuid references public.books(id) on delete cascade,
  content text not null,
  embedding vector(3072),
  created_at timestamptz default now()
);

5. TABLAS DE VERSIONES

5.1 versions

create table public.versions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  book_id uuid references public.books(id) on delete cascade,
  chapter_id uuid references public.chapters(id) on delete set null,
  parent_version_id uuid references public.versions(id) on delete set null,
  title text,
  prompt_used text,
  mode text,
  variables jsonb,
  generated_content text not null,
  visibility text default 'private',
  status text default 'active',
  is_beta boolean default false,
  created_at timestamptz default now()
);

Modos:
- literary
- systemic_analysis
- cinematic_variation
- full_novel
- child_version
- producer_pitch
- editor_review

5.2 version_embeddings

create table public.version_embeddings (
  id uuid primary key default gen_random_uuid(),
  version_id uuid references public.versions(id) on delete cascade,
  book_id uuid references public.books(id) on delete cascade,
  content text not null,
  embedding vector(3072),
  created_at timestamptz default now()
);

6. TABLAS DE VARIABLES CREATIVAS

6.1 creative_variables

create table public.creative_variables (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  category text,
  is_active boolean default true
);

Variables iniciales:
- immortality
- power
- artificial_intelligence
- religion
- love
- justice
- war
- memory
- betrayal
- family
- forgiveness
- ambition
- fear
- freedom

6.2 version_variable_values

create table public.version_variable_values (
  id uuid primary key default gen_random_uuid(),
  version_id uuid references public.versions(id) on delete cascade,
  variable_id uuid references public.creative_variables(id) on delete cascade,
  intensity int check (intensity >= 0 and intensity <= 100),
  notes text
);

7. TABLAS DE RANKING Y ENGAGEMENT

7.1 version_votes

create table public.version_votes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  version_id uuid references public.versions(id) on delete cascade,
  vote_type text check (vote_type in ('like', 'dislike')),
  created_at timestamptz default now(),
  unique(user_id, version_id)
);

7.2 version_comments

create table public.version_comments (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  version_id uuid references public.versions(id) on delete cascade,
  comment text not null,
  created_at timestamptz default now()
);

7.3 version_metrics

create table public.version_metrics (
  version_id uuid primary key references public.versions(id) on delete cascade,
  views_count int default 0,
  likes_count int default 0,
  forks_count int default 0,
  score numeric default 0,
  updated_at timestamptz default now()
);

8. TABLAS BETA

8.1 beta_features

create table public.beta_features (
  id uuid primary key default gen_random_uuid(),
  feature_key text unique not null,
  feature_name text not null,
  description text,
  status text default 'beta',
  visible_to_jury boolean default true,
  created_at timestamptz default now()
);

Features BETA:
- version_tree
- fork_of_fork
- specialized_reviewer
- child_line
- producer_mode
- publisher_dashboard
- marketplace
- subscription_engine
- ai_video_adaptation
- n8n_marketing_automation
- openclaw_commercial_agents

8.2 beta_waitlist

create table public.beta_waitlist (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  feature_key text,
  interest_level int default 5,
  notes text,
  created_at timestamptz default now()
);

9. TABLAS DE SUSCRIPCIÓN Y MONETIZACIÓN

9.1 plans

create table public.plans (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  price_monthly numeric,
  max_versions_per_month int,
  max_books int,
  features jsonb,
  is_active boolean default true
);

Planes sugeridos:
- Free Reader
- Creator Basic
- Creator Pro
- Author Studio
- Publisher / Producer Access

9.2 user_subscriptions

create table public.user_subscriptions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  plan_id uuid references public.plans(id),
  status text default 'trial',
  started_at timestamptz default now(),
  ends_at timestamptz
);

9.3 usage_limits

create table public.usage_limits (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  month text,
  versions_created int default 0,
  gemini_requests int default 0,
  embeddings_created int default 0,
  updated_at timestamptz default now()
);

10. TABLAS PARA PROMPTS

10.1 prompt_templates

create table public.prompt_templates (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  mode text,
  system_prompt text not null,
  user_prompt_template text not null,
  is_active boolean default true,
  created_at timestamptz default now()
);

Prompts iniciales:
- Generate Literary Variation
- Generate Systemic Analysis
- Generate Cinematic Version
- Generate Full Novel Expansion
- General Reviewer
- Specialized Reviewer
- Child Adaptation
- Producer Pitch

11. TABLAS DE AUDITORÍA

11.1 ai_requests

create table public.ai_requests (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id),
  provider text default 'gemini',
  model text,
  request_type text,
  input_summary text,
  output_summary text,
  tokens_input int,
  tokens_output int,
  status text,
  error_message text,
  created_at timestamptz default now()
);

11.2 audit_logs

create table public.audit_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id),
  action text not null,
  table_name text,
  record_id uuid,
  metadata jsonb,
  created_at timestamptz default now()
);

12. STORAGE EN SUPABASE

Crear buckets:

- book-covers
- author-assets
- user-uploads
- generated-assets
- beta-demo-assets

Reglas:
- book-covers: lectura pública, escritura autenticada.
- user-uploads: privado por usuario.
- generated-assets: lectura según visibilidad de versión.
- beta-demo-assets: visible para demo/jurado.

13. FUNCIONES RPC PARA BÚSQUEDA SEMÁNTICA

13.1 Buscar capítulos similares

create or replace function public.match_chapters (
  query_embedding vector(3072),
  match_threshold float,
  match_count int
)
returns table (
  chapter_id uuid,
  book_id uuid,
  content text,
  similarity float
)
language sql stable
as $$
  select
    chapter_embeddings.chapter_id,
    chapter_embeddings.book_id,
    chapter_embeddings.content,
    1 - (chapter_embeddings.embedding <=> query_embedding) as similarity
  from public.chapter_embeddings
  where 1 - (chapter_embeddings.embedding <=> query_embedding) > match_threshold
  order by chapter_embeddings.embedding <=> query_embedding
  limit match_count;
$$;

13.2 Buscar versiones similares

create or replace function public.match_versions (
  query_embedding vector(3072),
  match_threshold float,
  match_count int
)
returns table (
  version_id uuid,
  book_id uuid,
  content text,
  similarity float
)
language sql stable
as $$
  select
    version_embeddings.version_id,
    version_embeddings.book_id,
    version_embeddings.content,
    1 - (version_embeddings.embedding <=> query_embedding) as similarity
  from public.version_embeddings
  where 1 - (version_embeddings.embedding <=> query_embedding) > match_threshold
  order by version_embeddings.embedding <=> query_embedding
  limit match_count;
$$;

14. ÍNDICES RECOMENDADOS

create index chapters_book_id_idx on public.chapters(book_id);
create index versions_user_id_idx on public.versions(user_id);
create index versions_book_id_idx on public.versions(book_id);
create index versions_parent_version_id_idx on public.versions(parent_version_id);
create index version_votes_version_id_idx on public.version_votes(version_id);
create index ai_requests_user_id_idx on public.ai_requests(user_id);

Índices vectoriales:

create index chapter_embeddings_embedding_idx
on public.chapter_embeddings
using ivfflat (embedding vector_cosine_ops)
with (lists = 100);

create index version_embeddings_embedding_idx
on public.version_embeddings
using ivfflat (embedding vector_cosine_ops)
with (lists = 100);

15. ROW LEVEL SECURITY

Activar RLS:

alter table public.profiles enable row level security;
alter table public.books enable row level security;
alter table public.chapters enable row level security;
alter table public.versions enable row level security;
alter table public.version_votes enable row level security;
alter table public.version_comments enable row level security;
alter table public.user_subscriptions enable row level security;
alter table public.usage_limits enable row level security;

Política profiles:

create policy "Users can view own profile"
on public.profiles for select
using (auth.uid() = id);

create policy "Users can update own profile"
on public.profiles for update
using (auth.uid() = id);

Política books públicos:

create policy "Anyone can view public books"
on public.books for select
using (visibility = 'public');

create policy "Creators can insert books"
on public.books for insert
with check (auth.uid() = created_by);

Política versions:

create policy "Users can view own private versions"
on public.versions for select
using (
  visibility = 'public'
  or auth.uid() = user_id
);

create policy "Users can create own versions"
on public.versions for insert
with check (auth.uid() = user_id);

create policy "Users can update own versions"
on public.versions for update
using (auth.uid() = user_id);

16. EDGE FUNCTIONS RECOMENDADAS

Crear Edge Functions:

1. generate-version
Función:
- Recibe book_id, chapter_id, mode, variables, user_prompt.
- Recupera contenido base.
- Genera embedding del input con Gemini.
- Busca contexto similar en Supabase RPC.
- Construye prompt final.
- Llama Gemini.
- Guarda versión.
- Genera embedding de la nueva versión.
- Guarda version_embedding.
- Registra ai_request.

2. generate-embedding
Función:
- Recibe texto.
- Llama Gemini embedding.
- Devuelve vector.

3. review-version
Función:
- Recibe version_id.
- Usa Gemini para evaluar:
  - calidad narrativa
  - coherencia
  - originalidad
  - potencial editorial
  - potencial cinematográfico
  - riesgos de copyright
  - sugerencias

4. create-version-fork
Función:
- Crea una nueva versión basada en otra versión.
- Mantiene parent_version_id.
- Permite árbol de versiones.

5. beta-feature-interest
Función:
- Guarda interés del usuario en una función BETA.

17. ESTRUCTURA DEL FLUJO IA

Flujo para generar una versión:

Frontend Lovable:
- Usuario selecciona libro.
- Usuario selecciona capítulo.
- Usuario escoge modo.
- Usuario ajusta variables.
- Usuario hace clic en Generate.

Supabase Edge Function:
- Valida usuario.
- Revisa límite mensual.
- Recupera capítulo.
- Genera embedding del prompt con Gemini.
- Busca capítulos/versiones similares.
- Construye contexto.
- Llama Gemini.
- Guarda resultado.
- Guarda embedding.
- Actualiza usage_limits.
- Devuelve versión al frontend.

18. CÓDIGO BASE PYTHON PARA GEMINI + SUPABASE

Este ejemplo sustituye OpenAI por Gemini:

from google import genai
from supabase import create_client, Client

GEMINI_API_KEY = "API_KEY_GEMINI"
SUPABASE_URL = "URL_SUPABASE"
SUPABASE_KEY = "SERVICE_ROLE_KEY_SUPABASE"

client = genai.Client(api_key=GEMINI_API_KEY)
supabase: Client = create_client(SUPABASE_URL, SUPABASE_KEY)

persona = {
    "nombre": "Ana",
    "apellido": "Gómez",
    "mensaje": "Hola, este es un mensaje de prueba para generar embeddings."
}

insert_result = supabase.table("mensajes").insert(persona).execute()
mensaje_id = insert_result.data[0]["id"]

embedding_result = client.models.embed_content(
    model="gemini-embedding-001",
    contents=persona["mensaje"]
)

embedding = embedding_result.embeddings[0].values

supabase.table("mensaje_embeddings").insert({
    "mensaje_id": mensaje_id,
    "embedding": embedding
}).execute()

print("Embedding generado con Gemini y guardado en Supabase.")

19. ESTRUCTURA RECOMENDADA PARA LOVABLE

Lovable debe saber que el backend ya tendrá:

- auth.users
- profiles
- books
- chapters
- versions
- creative_variables
- version_votes
- version_comments
- version_metrics
- beta_features
- plans
- user_subscriptions
- usage_limits

Y debe llamar:
- generate-version
- review-version
- create-version-fork
- match_chapters
- match_versions

20. DATOS SEED PARA DEMO DEL JURADO

Insertar:

- 2 libros demo.
- 5 capítulos demo.
- 10 variables creativas.
- 3 planes.
- 8 beta_features.
- 3 prompt_templates.
- 1 versión pública demo.
- 1 árbol de versiones simple.

21. QUÉ MOSTRAR COMO MVP

MVP real:
- Login.
- Biblioteca.
- Selección de capítulo.
- Variables creativas.
- Generar versión con Gemini.
- Guardar versión.
- Ver mis versiones.
- Publicar versión.
- Ranking simple.
- Búsqueda semántica.

22. QUÉ MOSTRAR COMO BETA

BETA visible:
- Version Tree.
- Producer Mode.
- Publisher Marketplace.
- Child Line.
- Specialized Reviewer.
- Subscription Dashboard.
- Video Adaptation.
- n8n Marketing Automation.
- OpenClaw Commercial Agents.

Estas funciones pueden estar visibles con etiquetas:
- BETA
- Coming soon
- Request access
- Join waitlist

23. DECISIÓN CLAVE PARA EL HACKATHON

Para el Gemini XPRIZE Hackathon:

NO:
- No presentar OpenAI como motor central.
- No usar embeddings de OpenAI.
- No depender de ChatGPT para la generación visible.

SÍ:
- Gemini genera embeddings.
- Gemini genera versiones.
- Gemini revisa versiones.
- Gemini explica diferencias.
- Gemini ayuda a convertir libros en universos narrativos.
- Supabase organiza, guarda, protege y permite búsqueda vectorial.

24. RESUMEN FINAL DE ARQUITECTURA

byDebut =
Lovable frontend
+ Supabase backend
+ PostgreSQL database
+ pgvector semantic memory
+ Supabase Auth
+ Supabase Storage
+ Supabase Edge Functions
+ Gemini embeddings
+ Gemini generative AI
+ n8n/OpenClaw como capa futura comercial
