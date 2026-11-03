# Shahrazad Architecture Review V4.1 — التدقيق الشامل وبوابة الاعتماد

**الحالة: مسودة تدقيقية للمراجعة فقط (Review Only).**  
**تنبيه أمان الإنتاج:** لم يتم تشغيل أي أمر SQL على قاعدة بيانات الإنتاج (Production)، ولم يتم تعديل أي كود في التطبيق أو حذف أي بيانات.

---

## 1. الملخص التنفيذي وخلاصة الفحص الميداني

أظهر التدقيق المستقل للمشروع والملفات المتاحة ما يلي:

1. **سلامة الاتجاه المعماري الأساسي:**
   - الاعتماد على قاعدة البيانات كطابور مهام خلفي (Database-backed Job Runner) خيار سليم وموفر للتكاليف والتعقيد التشغيلي (لا حاجة لـ Redis أو خوادم منفصلة).
   - تقسيم النصوص إلى مشاهد دلالية (Semantic Segments) وتوليد السيناريو والصوت حسب لغة الواجهة المطلوبة فقط (On-demand Localization) خيار فعال.
   - الاعتماد على PostgreSQL Full-Text Search (FTS) دون Vector DB مناسب تماماً لحجم كتب الأطفال وللسياق المحدد.

2. **العوائق الحالية قبل اعتماد الـ Migration:**
   - **فجوة Schema:** ملفات الـ migrations الموجودة في مستودع التطبيق تنتهي عند `20261004212219_c5231154-2e51-4192-be11-6a042a6143e7.sql`. الجداول الحالية تشمل `books` و`pages` و`reading_sessions` و`reading_attempts` و`difficult_words` و`review_attempts`. لا يوجد في المستودع أي تعريف لـ `story_segments` أو `story_jobs` أو عمود `story_source_change_token` داخل `pages`.
   - **غياب الوصول لقاعدة Staging الحية:** لم تتصل أي أداة بقاعدة بيانات Supabase الحية؛ لذا لا يمكن الجزم بوجود أعمدة أو قيود إضافية تم تطبيقها يدوياً.
   - **ثغرات أمنية وتشغيلية في مسودة V4:**
     - لقطة التغيير `source_change_token_snapshot` كانت على مستوى المشهد ككل، بينما المشهد متعدد الصفحات يحتاج لقطة لكل صفحة داخل `story_segment_pages`.
     - سياسات RLS السابقة كانت تمنح المستخدم المصادق عليه `FOR ALL` على جداول الإكمال والجلسات، مما يتيح تسجيل إكمال وهمي دون استدعاء الـ RPCs.
     - عدم وجود تحقق ذري بين تقدم المشغل وصلاحية مصدر الصفحة وقت تسجيل الإكمال.
     - دالة `story_backfill_book` كانت تبحث عن أعلى صفحة مغطاة بدلاً من أول فجوة غير مجهزة.
     - دمج المهام (Job Coalescing) كان يكتفي بمرور استعلامي واحد ولا يحقق الإغلاق المتعدي الكامل (Transitive Closure).

---

## 2. مصفوفة التدقيق والتقييم التفصيلي (نقطة بنقطة)

| # | النقطة | التصنيف | التقييم والتعليل التقني |
|---|---|---|---|
| **1** | **Page Source Tokens & Segment Integrity** | **Not implemented (Blocked by missing live schema evidence)** | في V4، وُضِع Snapshot واحد على مستوى المشهد. إذا كان المشهد يغطي صفحتين (مثلاً صفحة 3 و4) وتم تعديل OCR للصفحة 4 فقط، تتغير لقطة الصفحة 4 بينما تبقى الصفحة 3 كما هي. المقارنة الفردية لا تكفي. التصميم الصحيح يتطلب حفظ `page_token_snapshot` داخل جدول الربط `story_segment_pages` لكل صفحة، وإبطال المشهد متى ما اختلف أي رمز لصفحاته. كما يجب التأكد أولاً من وجود العمود `pages.story_source_change_token` عبر استعلام الفحص قبل كتابة الـ Migration. |
| **2** | **Page Lifecycle Coverage** | **Implemented differently in part; Not implemented in part** | حالات `pages.status` الحالية في كود المستودع هي: `processing`, `ocr_review_required`, `ready`, `failed` (لا يوجد `unreadable` كقيمة قياسية في الـ Check Constraint). يجب أن يتجاهل المحرك الصفحات قيد المعالجة، ويطلق المهام فقط عند الانتقال إلى `ready`، ويبطل المقاطع المتأثرة فوراً عند تعديل OCR أو التحول إلى `failed`، دون الحاجة لإعادة معالجة الكتاب كاملاً. |
| **3** | **Playback & No Spoilers** | **Not implemented to required security bar** | منع الـ Spoilers هو جوهر التجربة. في V4، كان يمكن للعميل إرسال موضع استماع مصطنع أو تسجيل إكمال عبر استدعاء مباشر لجدول الإكمالات نظراً لسياسة RLS الفضفاضة. الحل: سحب صلاحيات الإدخال المباشر من المستخدمين، حصر الإكمال في RPC محصن يتحقق ذرياً من مطابقة المشهد الحالي، صلاحية لقطات الصفحات، والتسلسل المنطقي للاستماع. |
| **4** | **Security & RLS Policies** | **Implemented differently in part; Not implemented in part** | تفعيل RLS على جميع الجداول خطوة أساسية، لكن يجب: (أ) سحب صلاحيات `INSERT/UPDATE/DELETE` المباشرة من دور `authenticated` على جداول الإكمال والمهام وطابور التنظيف، (ب) قفل دوال الـ Worker لتعمل فقط بصلاحية `service_role` وسحب `PUBLIC EXECUTE` عنها، (ج) الاعتماد الحصري على `auth.uid()` وعدم قبول أي `p_user_id` خارجي، (د) تأمين ملفات الصوت في Storage Bucket خاص بروابط موقعة قصيرة الأجل (Signed URLs). |
| **5** | **Page Reordering & Job Coalescing** | **Not implemented to required guarantee** | الفحص كشف أن `pages` في المستودع عليها فهرس عادي `(book_id, page_number)` ولا يوجد Unique Constraint مسجل. لكن إن وجد قيد في الإنتاج، فإن التعديل المباشر قد يصطدم به. في إعادة الترتيب، يلزم قفل الكتاب، والتحقق من اكتمال المصفوفة، واستخدام إزاحة مؤقتة آمنة، وتعطيل الـ Triggers الداخلية مؤقتاً لتوليد Job موحد واحد. في Coalescing، يلزم تكرار حلقة الدمج حتى استقرار النطاق لضمان الإغلاق المتعدي الكامل وفصل اللغات وأنواع المهام. |
| **6** | **Backfill & Background Worker** | **Implemented differently in part; Incomplete evidence** | خوارزمية Backfill الحالية كانت تأخذ `MAX(page_to)`، مما يتجاوز أي فجوات غير مجهزة في المنتصف. التصحيح: البحث عن أول صفحة `ready` غير مغطاة بمشهد سليم، وتجهيز نافذة محددة (10–15 صفحة) لضمان عدم إرهاق المعالج وقابلية الاستئناف. دورة الـ Worker تحتاج آليات Claim / Renew / Finish / Retry دقيقة، وحفظ مهام تنظيف التخزين في جدول مستقل لا يتأثر بحذف الكتاب (`ON DELETE CASCADE`). |
| **7** | **Multilingual Localization** | **Not implemented (Not demonstrated in code)** | يجب أن يظل تقسيم المشاهد والحقائق موحداً ومحايداً للغة. طلب لغة جديدة للواجهة يولد فقط سجل Localization (سيناريو وصوت) لتلك اللغة دون المساس بالمشاهد الأصلية. عند سؤال شهرزاد، يبحث FTS في حقائق المشاهد المكتملة أياً كانت لغة استماعها، ويرد النموذج بلغة الواجهة الحالية دون حرق للأحداث اللاحقة. |
| **8** | **Gemini TTS & STT Verification** | **TTS API shape documented; Live integration blocked; STT unverified** | تؤكد وثائق Google الرسمية لموديلي `gemini-3.8-flash-tts` و`gemini-3.8-flash-lite-tts` ضرورة وضع النص المنطوق في `parts[].text`، بينما توجيهات الأداء واللهجة (Style/Direction) توضع في `parts[].speech_metadata.style`. يجب في الخادم التحقق من الـ MIME الحقيقي ومن ترويسة ملف الصوت (WAV/MP3). في STT، يتم فحص `MediaRecorder.isTypeSupported` على المتصفح واستخدام الصيغة المدعومة فعلياً مع معالجة واضحة لرفض إذن الميكروفون. |
| **9** | **Staging Tests & Evidence** | **Not implemented** | لم تُجر اختبارات قبول مكتملة على قاعدة بيانات Staging فعلية متصلة بالتطبيق. تم إجراء اختبار أولي لبناء الكائنات على محاكي PostgreSQL 17 محلي، لكنه لا يعد بديلاً عن الاختبار الفعلي. يُشترط اجتياز مصفوفة الاختبارات المحددة في هذا المستند قبل الموافقة على أي نشر للإنتاج. |

---

## 3. استعلامات فحص قاعدة البيانات (Read-Only Audit SQL)

> **توجيه هام:** شغّل هذه الاستعلامات على بيئة **Staging** أولاً عبر Supabase SQL Editor لمعرفة البنية الحقيقية المسجلة حالياً قبل تنفيذ أي تغيير. هذه الاستعلامات للقراءة فقط ولا تعدل أي بيانات.

```sql
-- ============================================================================
-- 1. كشف بنية جدولي books و pages وأي أعمدة مخصصة
-- ============================================================================
SELECT 
    table_name,
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name IN ('books', 'pages')
ORDER BY table_name, ordinal_position;

-- ============================================================================
-- 2. كشف القيود الفعلية (Constraints: Primary, Foreign, Unique, Check)
-- ============================================================================
SELECT 
    c.conrelid::regclass AS table_name,
    c.conname AS constraint_name,
    c.contype AS constraint_type,
    pg_get_constraintdef(c.oid) AS constraint_definition
FROM pg_constraint c
JOIN pg_namespace n ON n.oid = c.connamespace
WHERE n.nspname = 'public'
  AND c.conrelid::regclass::text IN ('books', 'pages')
ORDER BY table_name, constraint_type;

-- ============================================================================
-- 3. كشف الفهارس القائمة (Indexes)
-- ============================================================================
SELECT 
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public'
  AND tablename IN ('books', 'pages')
ORDER BY tablename, indexname;

-- ============================================================================
-- 4. كشف أي جداول أو وظائف سابقة خاصة بـ Shahrazad إن كانت موجودة
-- ============================================================================
SELECT 
    table_name 
FROM information_schema.tables 
WHERE table_schema = 'public' 
  AND table_name LIKE 'story_%';

SELECT 
    proname AS function_name,
    prosecdef AS is_security_definer,
    proargnames AS argument_names
FROM pg_proc p
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname = 'public'
  AND proname LIKE 'story_%';
```

---

## 4. المسودة الهيكلية المقترحة للـ Migration (للمراجعة والتدقيق)

تتضمن هذه المسودة المعالجات المعمارية التالية:
1. إضافة `story_source_change_token` إلى جدول `pages` بحذر وشرط عدم الوجود المسبق.
2. جدول `story_segments` ككيان دلالي يربط عدة صفحات.
3. جدول `story_segment_pages` مع حفظ `page_token_snapshot` لكل صفحة على حدة لمنع أي خلط بين الصفحات.
4. جدول `story_segment_localizations` للسيناريو والصوت والـ FTS vector بحسب اللغة.
5. جدول `story_segment_completions` مع حظر التعديل المباشر من المستخدمين وحصره في دوال النظام.
6. جدول `story_storage_cleanup_queue` المستقل لمنع فقدان مهام تنظيف الصوت عند حذف الكتاب.
7. دوال `RPC` آمنة للتحقق من الاستماع ومنع حرق الأحداث، مع دعم كامل للـ Triggers.

```sql
-- ============================================================================
-- SHAHRAZAD ARCHITECTURE MIGRATION V4.1 (DRAFT FOR REVIEW)
-- ============================================================================

-- 1. التأكد من وجود عمود التتبع في جدول الصفحات
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns 
    WHERE table_schema = 'public' AND table_name = 'pages' AND column_name = 'story_source_change_token'
  ) THEN
    ALTER TABLE public.pages ADD COLUMN story_source_change_token bigint NOT NULL DEFAULT 1;
  END IF;
END $$;

-- 2. جدول المقاطع الدلالية (Semantic Segments)
CREATE TABLE IF NOT EXISTS public.story_segments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_order int NOT NULL,
  source_page_from int NOT NULL,
  source_page_to int NOT NULL,
  facts_summary text NOT NULL DEFAULT '',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_story_segments_order UNIQUE (book_id, segment_order),
  CONSTRAINT chk_story_segments_pages CHECK (source_page_from <= source_page_to)
);

CREATE INDEX IF NOT EXISTS idx_story_segments_book_pages ON public.story_segments(book_id, source_page_from, source_page_to);

-- 3. جدول الربط متعدد الصفحات مع Snapshot مستقل لكل صفحة (Multi-page Integrity)
CREATE TABLE IF NOT EXISTS public.story_segment_pages (
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  page_id uuid NOT NULL REFERENCES public.pages(id) ON DELETE CASCADE,
  page_number int NOT NULL,
  page_token_snapshot bigint NOT NULL,
  PRIMARY KEY (segment_id, page_id)
);

CREATE INDEX IF NOT EXISTS idx_story_segment_pages_page ON public.story_segment_pages(page_id);

-- 4. جدول التوطين (السيناريو والصوت والـ FTS لكل لغة)
CREATE TABLE IF NOT EXISTS public.story_segment_localizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  language text NOT NULL CHECK (language IN ('ar', 'pl', 'en')),
  source_hash text NOT NULL,
  script_status text NOT NULL DEFAULT 'pending' CHECK (script_status IN ('pending', 'processing', 'ready', 'failed')),
  script_text text,
  audio_status text NOT NULL DEFAULT 'pending' CHECK (audio_status IN ('pending', 'processing', 'ready', 'failed')),
  audio_storage_path text,
  audio_duration_seconds numeric(6, 2),
  audio_mime_type text DEFAULT 'audio/wav',
  fts_doc tsvector,
  error_detail text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_story_localizations_lang UNIQUE (segment_id, language)
);

CREATE INDEX IF NOT EXISTS idx_story_loc_fts ON public.story_segment_localizations USING gin(fts_doc);

-- 5. جدول طابور المهام (Job Runner Queue)
CREATE TABLE IF NOT EXISTS public.story_jobs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  job_type text NOT NULL CHECK (job_type IN ('segment_range', 'generate_script', 'synthesize_audio')),
  language text CHECK (language IN ('ar', 'pl', 'en')),
  page_from int NOT NULL,
  page_to int NOT NULL,
  status text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'succeeded', 'failed')),
  priority int NOT NULL DEFAULT 10,
  payload jsonb NOT NULL DEFAULT '{}'::jsonb,
  retry_count int NOT NULL DEFAULT 0,
  max_retries int NOT NULL DEFAULT 3,
  locked_by text,
  locked_at timestamptz,
  lease_expires_at timestamptz,
  last_error text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_story_jobs_fetch ON public.story_jobs(status, priority DESC, created_at ASC)
WHERE status = 'pending';

-- 6. جدول طابور تنظيف التخزين (مستقل ولا يتبع CASCADE لحذف الكتاب)
CREATE TABLE IF NOT EXISTS public.story_storage_cleanup_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL,
  storage_path text NOT NULL,
  reason text NOT NULL,
  status text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'done', 'failed')),
  retry_count int NOT NULL DEFAULT 0,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 7. جداول جلسات الاستماع والإكمال (مع منع التعديل المباشر)
CREATE TABLE IF NOT EXISTS public.story_listening_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  current_segment_id uuid REFERENCES public.story_segments(id) ON DELETE SET NULL,
  language text NOT NULL CHECK (language IN ('ar', 'pl', 'en')),
  playback_position_seconds numeric(6, 2) NOT NULL DEFAULT 0,
  last_checkpoint_at timestamptz NOT NULL DEFAULT now(),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS public.story_segment_completions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  completed_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_user_segment_completed UNIQUE (user_id, segment_id)
);

CREATE INDEX IF NOT EXISTS idx_story_completions_lookup ON public.story_segment_completions(user_id, book_id);

-- ============================================================================
-- سياسات الأمان والحماية (RLS & Grants)
-- ============================================================================

ALTER TABLE public.story_segments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_segment_pages ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_segment_localizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_jobs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_storage_cleanup_queue ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_listening_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_segment_completions ENABLE ROW LEVEL SECURITY;

-- السماح بالقراءة فقط لمالك الكتاب
CREATE POLICY "Users read own story segments" ON public.story_segments
FOR SELECT TO authenticated
USING (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_segments.book_id AND b.user_id = auth.uid()));

CREATE POLICY "Users read own segment pages" ON public.story_segment_pages
FOR SELECT TO authenticated
USING (EXISTS (
  SELECT 1 FROM public.story_segments s
  JOIN public.books b ON b.id = s.book_id
  WHERE s.id = story_segment_pages.segment_id AND b.user_id = auth.uid()
));

CREATE POLICY "Users read own localizations" ON public.story_segment_localizations
FOR SELECT TO authenticated
USING (EXISTS (
  SELECT 1 FROM public.story_segments s
  JOIN public.books b ON b.id = s.book_id
  WHERE s.id = story_segment_localizations.segment_id AND b.user_id = auth.uid()
));

CREATE POLICY "Users read own completions" ON public.story_segment_completions
FOR SELECT TO authenticated
USING (user_id = auth.uid());

CREATE POLICY "Users read own listening sessions" ON public.story_listening_sessions
FOR SELECT TO authenticated
USING (user_id = auth.uid());

-- حظر الكتابة المباشرة من authenticated وحصرها في service_role ودوال النظام المحصنة
REVOKE INSERT, UPDATE, DELETE ON public.story_segment_completions FROM authenticated;
REVOKE INSERT, UPDATE, DELETE ON public.story_listening_sessions FROM authenticated;
REVOKE ALL ON public.story_jobs FROM authenticated;
REVOKE ALL ON public.story_storage_cleanup_queue FROM authenticated;

GRANT SELECT ON public.story_segments, public.story_segment_pages, public.story_segment_localizations,
  public.story_segment_completions, public.story_listening_sessions TO authenticated;

GRANT ALL ON ALL TABLES IN SCHEMA public TO service_role;

-- ============================================================================
-- الدوال والإجراءات الذكية المحصنة (Security Definer RPCs)
-- ============================================================================

-- دالة تسجيل إكمال الاستماع مع التحقق الصارم من عدم حرق الأحداث
CREATE OR REPLACE FUNCTION public.story_complete_playback(
  p_session_id uuid,
  p_segment_id uuid,
  p_position_seconds numeric
)
RETURNS boolean
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_user_id uuid := auth.uid();
  v_book_id uuid;
  v_duration numeric;
  v_is_stale boolean;
BEGIN
  IF v_user_id IS NULL THEN
    RAISE EXCEPTION 'Not authenticated';
  END IF;

  -- 1. التحقق من ملكية الجلسة ومطابقة المشهد
  SELECT book_id INTO v_book_id
  FROM public.story_listening_sessions
  WHERE id = p_session_id AND user_id = v_user_id;

  IF v_book_id IS NULL THEN
    RAISE EXCEPTION 'Invalid listening session';
  END IF;

  -- 2. التحقق من سلامة وصلاحية المشهد (عدم تغير Token لأي من صفحاته)
  SELECT EXISTS (
    SELECT 1 
    FROM public.story_segment_pages ssp
    JOIN public.pages p ON p.id = ssp.page_id
    WHERE ssp.segment_id = p_segment_id
      AND (p.status != 'ready' OR p.story_source_change_token != ssp.page_token_snapshot)
  ) INTO v_is_stale;

  IF v_is_stale THEN
    RAISE EXCEPTION 'Segment source content has changed; cannot complete stale playback';
  END IF;

  -- 3. التحقق من مدة الصوت الفعلي لمنع القفز المباشر لنهاية المقطع
  SELECT audio_duration_seconds INTO v_duration
  FROM public.story_segment_localizations
  WHERE segment_id = p_segment_id AND audio_status = 'ready'
  LIMIT 1;

  IF v_duration IS NOT NULL AND v_duration > 0 THEN
    IF p_position_seconds < (v_duration * 0.85) THEN
      RAISE EXCEPTION 'Playback position (% s) has not reached completion threshold for segment (% s)', p_position_seconds, v_duration;
    END IF;
  END IF;

  -- 4. تسجيل الإكمال بنجاح وبشكل ذري
  INSERT INTO public.story_segment_completions (user_id, book_id, segment_id)
  VALUES (v_user_id, v_book_id, p_segment_id)
  ON CONFLICT (user_id, segment_id) DO NOTHING;

  RETURN true;
END;
$$;

-- دالة البحث في المشاهد المكتملة فقط للإجابة عن أسئلة الطفلة دون Spoilers
CREATE OR REPLACE FUNCTION public.story_search_completed_segments(
  p_book_id uuid,
  p_query text,
  p_limit int DEFAULT 3
)
RETURNS TABLE (
  segment_id uuid,
  segment_order int,
  facts_summary text,
  script_text text,
  rank real
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_user_id uuid := auth.uid();
BEGIN
  IF v_user_id IS NULL THEN
    RAISE EXCEPTION 'Not authenticated';
  END IF;

  RETURN QUERY
  SELECT 
    s.id AS segment_id,
    s.segment_order,
    s.facts_summary,
    loc.script_text,
    ts_rank(loc.fts_doc, plainto_tsquery('arabic', p_query)) AS rank
  FROM public.story_segment_completions c
  JOIN public.story_segments s ON s.id = c.segment_id
  JOIN public.story_segment_localizations loc ON loc.segment_id = s.id
  WHERE c.user_id = v_user_id
    AND c.book_id = p_book_id
    AND loc.script_status = 'ready'
    AND loc.fts_doc @@ plainto_tsquery('arabic', p_query)
  ORDER BY rank DESC, s.segment_order DESC
  LIMIT p_limit;
END;
$$;

REVOKE ALL ON FUNCTION public.story_complete_playback(uuid, uuid, numeric) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION public.story_complete_playback(uuid, uuid, numeric) TO authenticated;

REVOKE ALL ON FUNCTION public.story_search_completed_segments(uuid, text, int) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION public.story_search_completed_segments(uuid, text, int) TO authenticated;
```

---

## 5. بوابة الاعتماد واختبارات القبول الإلزامية (Acceptance Test Gates)

للانتقال من مرحلة التخطيط إلى مرحلة التنفيذ الفعلي في بيئة Staging، يجب تشغيل واجتياز حزمة الاختبارات الآتية:

1. **اختبار العزل والحماية (RLS Isolation):**
   - محاولة مستخدم (User B) استدعاء `story_complete_playback` أو قراءة مقاطع كتاب يملكه مستخدم آخر (User A). يجب أن يفشل الطلب برمية خطأ واضحة أو نتيجة فارغة.
2. **اختبار المشاهد متعددة الصفحات (Multi-page Source Invalidation):**
   - إنشاء مشهد يعتمد على الصفحتين 1 و2 بـ Tokens مستقلين.
   - تعديل نص OCR للصفحة 2 فقط وتحديث Token الخاص بها.
   - التحقق من رفض `story_complete_playback` للمشهد القديم، ورفض توقيع رابط الصوت له، واستبعاده فوراً من نتائج البحث `story_search_completed_segments`.
3. **اختبار منع الحرق (Anti-Spoiler Gate):**
   - محاولة استعلام RAG عن شخصية أو حدث ورد في المشهد رقم 5 بينما الطفلة لم تكمل سوى المشهدين 1 و2.
   - التحقق من أن الاستعلام يرجع فقط حقائق المشهدين 1 و2 ويرفض أي سياق للمشاهد غير المكتملة.
4. **اختبار التعافي واستئناف العمليات (Worker Lease & Recovery):**
   - حجز مهمة بـ Lease مدته دقيقة واحدة.
   - محاكاة تعطل العامل وتجاوز مدة الـ Lease.
   - التحقق من أن عاملاً آخر يستطيع استرداد المهمة وتجديدها دون حدوث تكرار أو تعليق.
5. **اختبار سلامة التخزين (Storage Cleanup Queue Integrity):**
   - إبطال مقطع يحتوي على ملف صوتي مسجل في التخزين، أو حذف كتاب تجريبي.
   - التأكد من إدراج مسارات الملفات في `story_storage_cleanup_queue` وعدم حذف السجلات عشوائياً حتى يكتمل حذف الملف الحقيقي من الـ Bucket.

---

## 6. الخطوات التالية المقترحة

1. **تشغيل استعلامات الفحص (القسم 3)** على Supabase Staging وتزويدنا بمخرجات الأعمدة والقيود الفعلية لجدول `pages`.
2. **تطبيق Migration التجريبية في بيئة Staging فقط**.
3. **تشغيل حزمة اختبارات القبول (القسم 5)** واعتماد النتائج.
4. **الانتقال بعد ذلك مباشرة لتوصيل موديلي Gemini TTS و STT والواجهة**.
