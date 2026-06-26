Copia y pega en **Supabase SQL Editor** en este orden.

## 1. Extensiones

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

## 2. ENUMS

```sql
DO $$ BEGIN
  CREATE TYPE app_role AS ENUM ('reader','creator','author','editor','publisher','admin');
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

DO $$ BEGIN
  CREATE TYPE feature_status AS ENUM ('mvp','beta','disabled');
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

DO $$ BEGIN
  CREATE TYPE generation_mode AS ENUM (
    'literary',
    'systemic_analysis',
    'cinematic_variation',
    'full_novel',
    'children_story'
  );
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

DO $$ BEGIN
  CREATE TYPE visibility_type AS ENUM (
    'private',
    'public',
    'community',
    'publisher_only'
  );
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

DO $$ BEGIN
  CREATE TYPE review_type AS ENUM (
    'general',
    'specialized',
    'literary',
    'commercial',
    'cinematic',
    'ethical',
    'children'
  );
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;

DO $$ BEGIN
  CREATE TYPE subscription_plan AS ENUM (
    'free',
    'reader_plus',
    'creator_pro',
    'author_partner',
    'publisher_partner'
  );
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;
```

## 3. Tablas principales

```sql
CREATE TABLE IF NOT EXISTS public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT,
  display_name TEXT,
  avatar_url TEXT,
  role app_role DEFAULT 'reader',
  plan subscription_plan DEFAULT 'free',
  credits_available INT DEFAULT 5,
  country TEXT,
  language_preference TEXT DEFAULT 'en',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.books (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  subtitle TEXT,
  author_name TEXT,
  description TEXT,
  genre TEXT,
  language TEXT DEFAULT 'en',
  cover_url TEXT,
  original_book_url TEXT,
  status feature_status DEFAULT 'mvp',
  visibility visibility_type DEFAULT 'private',
  is_official BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.chapters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id UUID REFERENCES public.books(id) ON DELETE CASCADE,
  chapter_number INT NOT NULL,
  title TEXT,
  summary TEXT,
  content TEXT NOT NULL,
  status feature_status DEFAULT 'mvp',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.scenes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id UUID REFERENCES public.books(id) ON DELETE CASCADE,
  chapter_id UUID REFERENCES public.chapters(id) ON DELETE CASCADE,
  scene_number INT,
  title TEXT,
  summary TEXT,
  location TEXT,
  time_period TEXT,
  emotional_tone TEXT,
  content TEXT NOT NULL,
  status feature_status DEFAULT 'beta',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.characters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id UUID REFERENCES public.books(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  role_in_story TEXT,
  description TEXT,
  motivation TEXT,
  conflict TEXT,
  arc_summary TEXT,
  status feature_status DEFAULT 'beta',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.scene_characters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scene_id UUID REFERENCES public.scenes(id) ON DELETE CASCADE,
  character_id UUID REFERENCES public.characters(id) ON DELETE CASCADE,
  importance_level INT DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 4. Variables narrativas

```sql
CREATE TABLE IF NOT EXISTS public.narrative_variables (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  category TEXT,
  default_value TEXT,
  status feature_status DEFAULT 'mvp',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO public.narrative_variables
(name, description, category, default_value, status)
VALUES
('immortality', 'Explores the impact of immortality on human behavior.', 'philosophical', 'off', 'mvp'),
('power', 'Changes the role of power in the story.', 'political', 'medium', 'mvp'),
('love', 'Modifies romantic or emotional intensity.', 'emotional', 'medium', 'mvp'),
('justice', 'Changes the moral justice framework.', 'ethical', 'medium', 'mvp'),
('war', 'Increases or reduces conflict and war tension.', 'geopolitical', 'low', 'beta'),
('religion', 'Introduces religious or spiritual interpretation.', 'philosophical', 'off', 'beta'),
('memory', 'Changes how memory affects identity and decisions.', 'psychological', 'medium', 'beta'),
('artificial_intelligence', 'Changes the role of AI in the story.', 'technological', 'medium', 'mvp')
ON CONFLICT DO NOTHING;
```

## 5. Versiones, reviews, ranking y créditos

```sql
CREATE TABLE IF NOT EXISTS public.story_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_version_id UUID REFERENCES public.story_versions(id) ON DELETE SET NULL,
  book_id UUID REFERENCES public.books(id) ON DELETE CASCADE,
  chapter_id UUID REFERENCES public.chapters(id) ON DELETE SET NULL,
  scene_id UUID REFERENCES public.scenes(id) ON DELETE SET NULL,
  creator_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  prompt TEXT NOT NULL,
  generated_content TEXT NOT NULL,
  generation_mode generation_mode DEFAULT 'literary',
  visibility visibility_type DEFAULT 'private',
  status feature_status DEFAULT 'mvp',
  is_fork BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.version_variables (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version_id UUID REFERENCES public.story_versions(id) ON DELETE CASCADE,
  variable_id UUID REFERENCES public.narrative_variables(id) ON DELETE CASCADE,
  selected_value TEXT,
  weight NUMERIC DEFAULT 1.0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.ai_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version_id UUID REFERENCES public.story_versions(id) ON DELETE CASCADE,
  reviewer_type review_type DEFAULT 'general',
  score NUMERIC,
  strengths TEXT,
  weaknesses TEXT,
  commercial_potential TEXT,
  literary_quality TEXT,
  cinematic_potential TEXT,
  ethical_notes TEXT,
  recommendation TEXT,
  raw_response JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.version_rankings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version_id UUID REFERENCES public.story_versions(id) ON DELETE CASCADE,
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  rating INT CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(version_id, user_id)
);

CREATE TABLE IF NOT EXISTS public.credit_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  amount INT NOT NULL,
  reason TEXT,
  related_version_id UUID REFERENCES public.story_versions(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 6. Features MVP/BETA

```sql
CREATE TABLE IF NOT EXISTS public.features (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  status feature_status DEFAULT 'mvp',
  frontend_visible BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO public.features
(name, description, status, frontend_visible)
VALUES
('Book Library', 'Users can browse available books.', 'mvp', true),
('Create Literary Variation', 'Users can generate alternate literary versions.', 'mvp', true),
('Variable Editor', 'Users can change narrative variables.', 'mvp', true),
('AI General Review', 'Gemini reviews the generated version.', 'mvp', true),
('Ranking System', 'Users can rate versions.', 'mvp', true),
('Semantic Search', 'Users can search by meaning using embeddings.', 'mvp', true),
('Scene-Level Forking', 'Users can fork specific scenes.', 'beta', true),
('Children Story Mode', 'Generate age-based children stories.', 'beta', true),
('Illustrated Children Books', 'Generate children books with illustrations.', 'beta', true),
('Cinematic Version Mode', 'Generate screenplay-like versions.', 'beta', true),
('Full Novel Expansion', 'Expand a chapter or scene into a complete novel.', 'beta', true),
('Publisher Dashboard', 'Publisher tools for book performance.', 'beta', true),
('Revenue Sharing', 'Revenue split among platform, author, editor, publisher.', 'beta', true),
('Producer Director Access', 'Access for producers and directors.', 'beta', true)
ON CONFLICT DO NOTHING;
```

## 7. Embeddings

```sql
CREATE TABLE IF NOT EXISTS public.embeddings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
  semantic_text TEXT NOT NULL,
  embedding VECTOR(768),
  embedding_model TEXT DEFAULT 'gemini-embedding',
  status feature_status DEFAULT 'mvp',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 8. Función de búsqueda semántica

```sql
CREATE OR REPLACE FUNCTION public.match_embeddings (
  query_embedding VECTOR(768),
  match_count INT DEFAULT 10,
  filter_entity_type TEXT DEFAULT NULL
)
RETURNS TABLE (
  id UUID,
  entity_type TEXT,
  entity_id UUID,
  semantic_text TEXT,
  similarity FLOAT
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    e.id,
    e.entity_type,
    e.entity_id,
    e.semantic_text,
    1 - (e.embedding <=> query_embedding) AS similarity
  FROM public.embeddings e
  WHERE filter_entity_type IS NULL
     OR e.entity_type = filter_entity_type
  ORDER BY e.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

## 9. Logs e invitaciones

```sql
CREATE TABLE IF NOT EXISTS public.ai_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
  function_name TEXT,
  model_name TEXT,
  input_tokens INT,
  output_tokens INT,
  cost_estimate NUMERIC,
  request_payload JSONB,
  response_payload JSONB,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.partner_invitations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL,
  partner_type app_role,
  invited_by UUID REFERENCES public.profiles(id) ON DELETE SET NULL,
  status TEXT DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 10. Índices

```sql
CREATE INDEX IF NOT EXISTS idx_books_owner ON public.books(owner_id);
CREATE INDEX IF NOT EXISTS idx_chapters_book ON public.chapters(book_id);
CREATE INDEX IF NOT EXISTS idx_scenes_book ON public.scenes(book_id);
CREATE INDEX IF NOT EXISTS idx_scenes_chapter ON public.scenes(chapter_id);
CREATE INDEX IF NOT EXISTS idx_characters_book ON public.characters(book_id);
CREATE INDEX IF NOT EXISTS idx_story_versions_book ON public.story_versions(book_id);
CREATE INDEX IF NOT EXISTS idx_story_versions_creator ON public.story_versions(creator_id);
CREATE INDEX IF NOT EXISTS idx_embeddings_entity_type ON public.embeddings(entity_type);
CREATE INDEX IF NOT EXISTS idx_embeddings_entity_id ON public.embeddings(entity_id);

CREATE INDEX IF NOT EXISTS idx_embeddings_vector
ON public.embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

## 11. RLS

```sql
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.books ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chapters ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.scenes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.characters ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.scene_characters ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.narrative_variables ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.version_variables ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.ai_reviews ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.version_rankings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.credit_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.features ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.embeddings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.ai_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.partner_invitations ENABLE ROW LEVEL SECURITY;
```

## 12. Policies

```sql
DROP POLICY IF EXISTS "Users can view own profile" ON public.profiles;
CREATE POLICY "Users can view own profile"
ON public.profiles
FOR SELECT
USING (auth.uid() = id);

DROP POLICY IF EXISTS "Users can update own profile" ON public.profiles;
CREATE POLICY "Users can update own profile"
ON public.profiles
FOR UPDATE
USING (auth.uid() = id);

DROP POLICY IF EXISTS "Users can view own or public books" ON public.books;
CREATE POLICY "Users can view own or public books"
ON public.books
FOR SELECT
USING (
  owner_id = auth.uid()
  OR visibility IN ('public', 'community')
);

DROP POLICY IF EXISTS "Users can create own books" ON public.books;
CREATE POLICY "Users can create own books"
ON public.books
FOR INSERT
WITH CHECK (owner_id = auth.uid());

DROP POLICY IF EXISTS "Users can update own books" ON public.books;
CREATE POLICY "Users can update own books"
ON public.books
FOR UPDATE
USING (owner_id = auth.uid());

DROP POLICY IF EXISTS "Users can view chapters from accessible books" ON public.chapters;
CREATE POLICY "Users can view chapters from accessible books"
ON public.chapters
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM public.books b
    WHERE b.id = chapters.book_id
    AND (
      b.owner_id = auth.uid()
      OR b.visibility IN ('public', 'community')
    )
  )
);

DROP POLICY IF EXISTS "Users can view scenes from accessible books" ON public.scenes;
CREATE POLICY "Users can view scenes from accessible books"
ON public.scenes
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM public.books b
    WHERE b.id = scenes.book_id
    AND (
      b.owner_id = auth.uid()
      OR b.visibility IN ('public', 'community')
    )
  )
);

DROP POLICY IF EXISTS "Users can view characters from accessible books" ON public.characters;
CREATE POLICY "Users can view characters from accessible books"
ON public.characters
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM public.books b
    WHERE b.id = characters.book_id
    AND (
      b.owner_id = auth.uid()
      OR b.visibility IN ('public', 'community')
    )
  )
);

DROP POLICY IF EXISTS "Anyone can view narrative variables" ON public.narrative_variables;
CREATE POLICY "Anyone can view narrative variables"
ON public.narrative_variables
FOR SELECT
USING (true);

DROP POLICY IF EXISTS "Anyone can view visible features" ON public.features;
CREATE POLICY "Anyone can view visible features"
ON public.features
FOR SELECT
USING (frontend_visible = true);

DROP POLICY IF EXISTS "Users can view own or public versions" ON public.story_versions;
CREATE POLICY "Users can view own or public versions"
ON public.story_versions
FOR SELECT
USING (
  creator_id = auth.uid()
  OR visibility IN ('public', 'community')
);

DROP POLICY IF EXISTS "Users can create own versions" ON public.story_versions;
CREATE POLICY "Users can create own versions"
ON public.story_versions
FOR INSERT
WITH CHECK (creator_id = auth.uid());

DROP POLICY IF EXISTS "Users can update own versions" ON public.story_versions;
CREATE POLICY "Users can update own versions"
ON public.story_versions
FOR UPDATE
USING (creator_id = auth.uid());

DROP POLICY IF EXISTS "Users can view variables from own versions" ON public.version_variables;
CREATE POLICY "Users can view variables from own versions"
ON public.version_variables
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM public.story_versions sv
    WHERE sv.id = version_variables.version_id
    AND (
      sv.creator_id = auth.uid()
      OR sv.visibility IN ('public', 'community')
    )
  )
);

DROP POLICY IF EXISTS "Users can view reviews from accessible versions" ON public.ai_reviews;
CREATE POLICY "Users can view reviews from accessible versions"
ON public.ai_reviews
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM public.story_versions sv
    WHERE sv.id = ai_reviews.version_id
    AND (
      sv.creator_id = auth.uid()
      OR sv.visibility IN ('public', 'community')
    )
  )
);

DROP POLICY IF EXISTS "Users can view rankings" ON public.version_rankings;
CREATE POLICY "Users can view rankings"
ON public.version_rankings
FOR SELECT
USING (true);

DROP POLICY IF EXISTS "Users can create own rankings" ON public.version_rankings;
CREATE POLICY "Users can create own rankings"
ON public.version_rankings
FOR INSERT
WITH CHECK (user_id = auth.uid());

DROP POLICY IF EXISTS "Users can view own credit transactions" ON public.credit_transactions;
CREATE POLICY "Users can view own credit transactions"
ON public.credit_transactions
FOR SELECT
USING (user_id = auth.uid());

DROP POLICY IF EXISTS "Users can view embeddings from accessible content" ON public.embeddings;
CREATE POLICY "Users can view embeddings from accessible content"
ON public.embeddings
FOR SELECT
USING (true);
```

## 13. Storage buckets

```sql
INSERT INTO storage.buckets (id, name, public)
VALUES
('book-covers', 'book-covers', true),
('original-books', 'original-books', false),
('generated-assets', 'generated-assets', false),
('children-illustrations', 'children-illustrations', false),
('publisher-assets', 'publisher-assets', false)
ON CONFLICT (id) DO NOTHING;
```

## 14. Trigger para crear perfil automático

```sql
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, display_name)
  VALUES (
    NEW.id,
    NEW.raw_user_meta_data->>'full_name',
    NEW.raw_user_meta_data->>'display_name'
  )
  ON CONFLICT (id) DO NOTHING;

  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW
EXECUTE FUNCTION public.handle_new_user();
```

Listo. Después de esto, el siguiente paso sería crear las **Edge Functions** en Supabase.
