# Shahrazad Simplified Architecture & Staging Baseline Specification (V2)

**Document Version:** 2.0 (Streamlined Operational Scope)
**Audit Execution Reference:** 2026-11-03T17:40:55Z (PostgreSQL 17.6 Baseline)
**Project Reference:** Unconfirmed (Awaiting Supabase Dashboard link)
**Status:** Approved Simplified Architecture for Staging Execution

---

## Executive Summary & Streamlining Decisions

Following architectural reviews and explicit product directives, the scope of «احكي يا شهرزاد» is simplified to prioritize an immediate, robust, and cost-effective end-to-end user experience on Staging.

### 1. Simplified Handling of Page Modifications & Deletions
- **No Complex Historical Versioning**: Modifying OCR or deleting pages is an infrequent action in real-world usage.
- **Atomic State Invalidation**: When any page is edited or deleted, the book's story state is marked `stale` (`is_stale = true`).
- **Temporary Playback Pause**: The app immediately halts playback of outdated narrative scenes or obsolete question context until the book's story is re-prepared.
- **Progressive Re-preparation**: The storyteller scenes are re-generated in simple progressive batches.
- **Asynchronous Audio Cleanup**: Outdated audio files in Supabase Storage are enqueued for background cleanup without blocking the active reading or listening experience.

### 2. Core Functional Requirements (MVP)
1. **Progressive Background Preparation**: When new pages are photographed or uploaded, scenes are prepared incrementally in the background without forcing the child to wait for the entire book.
2. **Semantic Scene Division**: Pages are segmented into cohesive narrative scenes with beginning, development, and resolution, ensuring a lively and engaging storytelling rhythm.
3. **Language Matching UI**:
   - The story is narrated in the active UI language: **Egyptian Colloquial Arabic (العامية المصرية)**, **Polish**, or **English**.
   - Scripts and audio are synthesized **strictly on-demand** for the active language.
   - Generated audio is persisted and reused whenever the user returns to that language.
4. **Resilient Playback & State Persistence**: If the mobile browser closes, connection drops, or the child switches tabs, listening position (`last_position_seconds`) and active scene are persisted, allowing one-tap resumption.
5. **Interactive Push-to-Talk Voice Q&A**: The child presses a button, speaks her question in her voice, and Shahrazad answers warmly in the active UI language.
6. **Strict Anti-Spoiler Retrieval (No Spoilers)**: Shahrazad's Q&A answers are grounded **strictly in the scenes the child has fully completed listening to**. Unheard or future narrative segments are never passed to the LLM context.
7. **Automated Retry & Reliability**: Transient TTS or LLM generation failures automatically retry up to 3 times with exponential backoff without requiring parent or child intervention.
8. **Zero Operational Overhead**: Powered entirely by existing **PostgreSQL 17.6** and **Supabase Storage**. No Redis, Celery, VM, or external vector database.


---

## Section 1: Security & Privilege Remediation (Staging Findings)

The empirical read-only audit revealed that `anon` and `authenticated` roles possess excessive privileges (`TRUNCATE`, `TRIGGER`, `REFERENCES`) across all 9 public tables (`books`, `pages`, `reading_sessions`, etc.).

### Immediate Privilege Lockdown (Safe & Non-Breaking)
```sql
-- 1. Strip destructive TRUNCATE and administrative privileges from anon and authenticated
REVOKE TRUNCATE, TRIGGER, REFERENCES ON ALL TABLES IN SCHEMA public FROM anon, authenticated;

-- 2. Prevent anon access to private user data
REVOKE ALL ON TABLE public.books, public.pages, public.reading_sessions,
                     public.reading_attempts, public.review_attempts,
                     public.difficult_words, public.lesson_events,
                     public.settings, public.profiles FROM anon;

-- 3. Retain authenticated DML (SELECT, INSERT, UPDATE, DELETE) protected by RLS
-- Existing reading and capture functions remain completely unaffected.
```


---

## Section 2: Streamlined Schema Architecture (3 Minimal Tables)

To eliminate unnecessary complexity, the entire subsystem is structured into just **three minimal tables** and one tracking column on `public.books`:

### 2.1 Entity Relationship Overview
```
books (existing) ───────────┐ (1:N)
   │                        ▼
   │               public.story_scenes (Semantic scenes per book)
   │               ├── id, book_id, scene_order
   │               ├── title, start_page, end_page, source_hash
   │               └── status ('ready', 'stale')
   │                        │
   │                        ▼ (1:N)
   │               public.story_scene_audio (On-demand localized audio)
   │               ├── id, scene_id, book_id, language ('ar' | 'pl' | 'en')
   │               ├── script_text, audio_url, duration_seconds
   │               └── status ('pending', 'ready', 'failed')
   │
   └────────────────────────▼
                   public.story_playback_state (Child's listening & completion state)
                   ├── id, user_id, book_id, current_scene_id, language
                   ├── position_seconds, completed_scene_ids (uuid[])
                   └── updated_at
```

### 2.2 Complete, Minimal DDL Script
```sql
-- 1. Book-level story readiness & invalidation flag
ALTER TABLE public.books
  ADD COLUMN IF NOT EXISTS story_is_stale boolean NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS story_last_prepared_at timestamptz;

-- 2. Story Scenes (Semantic chunks)
CREATE TABLE IF NOT EXISTS public.story_scenes (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  scene_order integer NOT NULL,
  title text NOT NULL DEFAULT '',
  start_page_number integer NOT NULL,
  end_page_number integer NOT NULL,
  source_hash text NOT NULL,
  status text NOT NULL DEFAULT 'ready' CHECK (status IN ('ready', 'stale')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_book_scene_order UNIQUE (book_id, scene_order)
);

-- 3. Localized Scripts & Cached Audio
CREATE TABLE IF NOT EXISTS public.story_scene_audio (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  scene_id uuid NOT NULL REFERENCES public.story_scenes(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  language text NOT NULL CHECK (language IN ('ar', 'pl', 'en')),
  script_text text NOT NULL,
  summary_text text NOT NULL DEFAULT '',
  audio_url text,
  duration_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  status text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'ready', 'failed')),
  retry_count integer NOT NULL DEFAULT 0,
  last_error text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_scene_language UNIQUE (scene_id, language)
);

-- 4. Playback State & Completed Scenes (No-Spoilers Source)
CREATE TABLE IF NOT EXISTS public.story_playback_state (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  language text NOT NULL DEFAULT 'ar',
  current_scene_id uuid REFERENCES public.story_scenes(id) ON DELETE SET NULL,
  position_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  completed_scene_ids uuid[] NOT NULL DEFAULT '{}'::uuid[],
  last_played_at timestamptz NOT NULL DEFAULT now(),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_user_book_lang_playback UNIQUE (user_id, book_id, language)
);

-- 5. RLS Protection
ALTER TABLE public.story_scenes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_scene_audio ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_playback_state ENABLE ROW LEVEL SECURITY;

CREATE POLICY "own scenes" ON public.story_scenes
  FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_scenes.book_id AND b.user_id = auth.uid()))
  WITH CHECK (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_scenes.book_id AND b.user_id = auth.uid()));

CREATE POLICY "own scene audio" ON public.story_scene_audio
  FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_scene_audio.book_id AND b.user_id = auth.uid()))
  WITH CHECK (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_scene_audio.book_id AND b.user_id = auth.uid()));

CREATE POLICY "own playback state" ON public.story_playback_state
  FOR ALL TO authenticated
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());
```


---

## Section 3: Semantic Scene Division & Progressive Preparation

### 3.1 Progressive Grouping Rules
- A story scene is formed whenever **2 to 5 consecutive pages** are in `'ready'` status.
- Rather than waiting for the entire book to be uploaded, the worker clusters available ordered pages into self-contained scenes.
- A semantic scene prompt directs Gemini to synthesize the pages into a lively, narrative episode:
  - **Egyptian Arabic**: "احكي الحكاية كحكواتية مصرية بأسلوب مشوّق ولطيف لطفلة صغيرة، مع الحفاظ الدقيق على أسماء الشخصيات والأحداث دون أي اختلاق."
  - **Polish**: "Opowiedz tę historię ciepłym, żywym głosem dla małego dziecka, zachowując dokładnie imiona i fakty."
  - **English**: "Retell this story warmly and expressively for a young child, strictly adhering to the facts."


---

## Section 4: Strict No-Spoilers Q&A Protocol

When the child holds the microphone button and asks a question:
1. **Audio / Speech Recognition**: Captures the question in the active language.
2. **Context Assembly**:
   - Queries `story_playback_state` for `completed_scene_ids`.
   - If `completed_scene_ids` is empty (the child has not finished scene 1), only the current scene up to `position_seconds` or a gentle encouragement is allowed.
   - Fetches `script_text` and `summary_text` **only for scenes present in `completed_scene_ids`**.
   - Scenes ahead in the book are strictly omitted from the system prompt.
3. **Response Generation**:
   - Shahrazad answers in Egyptian Arabic (or the selected UI language) using solely the completed scenes.
   - If asked about future events, Shahrazad replies warmly: *"لسه ميعاد الحكاية دي ما جاش يا بطلة، تعالي نسمع ونعرف سوا!"*


---

## Section 5: Acceptance Test Checklist (Staging PASS/FAIL)

| Test | Objective | Pass Criteria |
| :--- | :--- | :--- |
| **ST-01: Table Creation** | Run simplified DDL | 3 new tables created; 0 errors. |
| **ST-02: Privilege Confinement** | `TRUNCATE public.books` by anon/auth | Denied (`42501 permission denied`). |
| **ST-03: Progressive Scene Generation** | Add 2 pages to a book | Scene 1 generated without waiting for full book. |
| **ST-04: Audio On-Demand** | Request Arabic playback | Arabic audio generated; Polish remains pending until requested. |
| **ST-05: Playback Resume** | Pause at 45s, reload page | Resumes exactly at 45s. |
| **ST-06: No-Spoilers Guard** | Ask question before Scene 2 is completed | Context includes Scene 1 only; zero Scene 2 details. |
| **ST-07: Invalidation Flag** | Edit page OCR text | `books.story_is_stale` is set to `true`; UI alerts child. |
