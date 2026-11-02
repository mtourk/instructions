# احكي يا شهرزاد: المعمارية وخطة التنفيذ وMigration للمراجعة

**الحالة:** تصميم للمراجعة فقط. لا يوجد تنفيذ أو تشغيل SQL ضمن هذا المستند.

## الخلاصة التنفيذية

الفكرة قابلة للتنفيذ، وهي مناسبة لكتاب قد يصل إلى 300 صفحة بشرط عدم التعامل معه كطلب واحد طويل. نعالج كل صفحة ثم مقاطعها كوحدات مستقلة، ونجهز النص والصوت في الخلفية على دفعات. عند التشغيل نقرأ المقاطع بالترتيب ونحمّل الصوت الحالي والتالي فقط. الأسئلة لا تستخدم إلا المقاطع التي اكتمل الاستماع إليها، منعاً للحرق أو التخمين.

أوصي في الإصدار الأول بـ **جدول وظائف دائم داخل Postgres مع Atomic Claiming وLease وRetry، ومجدول يستدعي Edge Function قصيرة كل دقيقة**. لا Redis، ولا VM، ولا سيرفر مستقل، ولا اعتماد على بقاء شاشة المتصفح مفتوحة. أفضّل هذا على PGMQ في البداية لأن جدول الوظائف نفسه يعطي حالة واضحة للأهل والتشخيص، ويتجنب ازدواج حالة التطبيق والرسائل. PGMQ بديل جيد ومدعوم كطابور Postgres إذا أثبت فحص قاعدة الإنتاج أنه مفعّل وأن تكامله أنسب؛ لكنه ليس شرطاً لهذا التصميم. [توثيق PGMQ](https://supabase.com/docs/guides/queues/pgmq)

## ما راجعته وما لا يزال غير مؤكد

- راجعت كود TTS الفعلي: المسار المباشر لـ Gemini يأخذ PCM ويغلّفه WAV مع معدل العينة الوارد، أما مسار Lovable Gateway فيعيد البايتات دون فحص MIME أو تحويل حاويتها. مع ذلك، واجهة التشغيل تنشئ Blob على أنه WAV. لذلك لا يصح تخزين خرج المسارين على افتراض أنه WAV قبل إضافة التحقق والتطبيع.
- الكود يعرّف نمطاً بولندياً، مصرياً وإنجليزياً، لكن واجهة `SpeechLang` الحالية لا تقبل إلا `pl-PL` و`ar-EG`. وجود نمط إنجليزي وحده لا يثبت أن كل مسارات الصوت تدعم الإنجليزية. يلزم اختبار فعلي لكل لغة ولكل موفر، بما فيه الحروف البولندية والتشكيل/النطق واللهجة المصرية.
- المشروع أبلغ عن قاعدة Lovable Cloud/Postgres عبر Supabase. محاولة القراءة المباشرة من قاعدة البيانات الحية فشلت برسالة أنها غير متاحة؛ لذلك لم أتحقق من امتدادات الإنتاج أو من تطابق الـschema الحي مع ملفات migrations. ملف SQL أدناه مسودة مراجعة مبنية على ملفات المشروع، وليست تأكيداً أنها جاهزة للصق في الإنتاج قبل المقارنة الحية.

## المعمارية المقترحة

### 1. استقبال الصفحة والتحضير الخلفي

1. ترفع الطفلة صورة الصفحة أو تحفظ نص OCR المعدّل كما يعمل التطبيق الآن.
2. عند جاهزية الصفحة، ينشئ Trigger وظيفة `process_page` تلقائياً في `story_jobs`. لا يحتاج المتصفح أن يبقى مفتوحاً.
3. العامل المجدول يطالب بوظائف باستخدام `SELECT … FOR UPDATE SKIP LOCKED` داخل تحديث ذري؛ يسجل `lease_token` ووقت انتهاء الـlease وعدد المحاولات.
4. يقسم النص ويستخرج الحقائق والأسماء والأحداث بترتيبها، ثم ينشئ مقاطع ثابتة الهوية. بعد ذلك تُنشأ وظيفة إعداد لكل مقطع/لغة عند الحاجة.
5. تنفذ كل وظيفة خطوة محدودة؛ لا تحاول توليد كتاب من 300 صفحة في HTTP request واحد. عند الإغلاق أو انقطاع العامل تنتهي الـlease، ثم تعود الوظيفة للطابور تلقائياً. أخطاء المزود تعاد بمحاولات متباعدة، وبعد الحد الأقصى تصبح `failed` مع رسالة تشخيصية قابلة لإعادة المحاولة.
6. عند إضافة صفحات متتالية، يمكن تأخير تجميع الدفعة فترة قصيرة أو تقسيمها إلى مجموعات، لكن لا ننتظر خمس صفحات كشرط. الصفحة الأولى تبدأ التحضير، والصفحات التالية تضاف بالتوازي المحدود.

### 2. حدود التشغيل: Supabase وCloudflare

لا يوجد حد HTTP عام ثابت من 30 أو 60 ثانية يمكن افتراضه لكل Worker. توثق Cloudflare حد CPU افتراضياً 30 ثانية في الاستخدام المدفوع، مع سقف قابل للتهيئة يصل إلى 5 دقائق، كما تذكر سقفاً أعلى لجدولة Cron/مستهلك Queue؛ هذا حد CPU وليس قاعدة تقول إن كل طلب HTTP ينتهي بعد 30 ثانية. كما توثق Supabase Edge Functions مدة حائطية 150 ثانية للخطة المجانية و400 ثانية للمدفوعة، وCPU قدره ثانيتان لكل طلب، وIdle timeout قدره 150 ثانية. `waitUntil` يمدد عمر عمل تابع للطلب لكنه ليس طابوراً دائماً. لذلك الوظائف الدائمة مع جدولة/عامل منفصل هي أساس الاستمرارية، لا HTTP طويل ولا `waitUntil`. [Cloudflare Workers limits](https://developers.cloudflare.com/workers/platform/limits/) · [Supabase Edge Function limits](https://supabase.com/docs/guides/functions/limits)

المقترح هو Supabase Cron/pg_net لاستدعاء Edge Function داخلية، بعد فحص الإضافات والصلاحيات على البيئة الحية. إذا كان النشر الفعلي يعتمد Cloudflare Cron بدلاً من Supabase Cron، يمكنه استدعاء العامل نفسه؛ لا نشغل مجدولين متنافسين. في الحالتين، قاعدة البيانات هي مصدر الحقيقة للطابور والـlease.

### 3. تقسيم وفهم القصة

- مصدر الحقيقة هو OCR/النص المحرر للصفحات، وليس سرداً حراً يطلب من Gemini اختراع كتاب كامل.
- تُقسم الصفحة إلى مقاطع متسلسلة قصيرة، وتُحفظ لكل مقطع: رقم ترتيب الصفحة، النص المصدر، بصمة المصدر، والأسماء/الأحداث/الحقائق المستخرجة مع الإشارة إلى الصفحة.
- يولّد النموذج نص حكاية مبسطاً للأطفال بلغة واجهة التطبيق، مع قواعد واضحة: أسماء الشخصيات والأحداث من المصدر فقط، والحفاظ على ترتيبها، وعدم إدخال شخصيات أو أحداث من خارج النص. نحتفظ بالحقائق المصدرية وبالنص المروي معاً كي يمكن تدقيقه.
- «احكي يا شهرزاد» يمر على المقاطع الجاهزة بترتيب الكتاب. لا يطلب من النموذج إعادة تأليف الكتاب كله في جلسة Live واحدة. المقاطعة/السؤال توقف التشغيل، يبحث الرد في المقاطع المكتملة فقط، ثم يكمل من المقطع نفسه/التالي وفق سياسة الاستئناف.
- الكتاب الكبير يُجهز تدريجياً على مستوى الصفحة/المقطع. يمكن للطفلة بدء الاستماع لما اكتمل، مع مؤشر جاهزية واضح لبقية الصفحات؛ لا يلزم انتظار انتهاء الكتاب كله.

### 4. Mini-RAG ومنع الحرق

- لا يدخل في البحث إلا مقطع له سجل `story_segment_listening.completed_at`، أي أن تشغيله انتهى فعلاً.
- لا يُستخدم نص المقطع الجاري إذا لم يُسجل اكتماله. إذا سألت الطفلة عن هذا المقطع ولا توجد بيانات جملة-بجملة عمّا سمعته، يوضح المساعد بلغة الواجهة أن الجزء لم يكتمل بعد، ويطلب منها إكماله أو يجيب من جزء مكتمل سابق ذي صلة. لا يخمّن ولا يكشف أحداثاً لاحقة.
- الإصدار الأول يستطيع البحث في حقائق/ملخصات المقاطع المكتملة مع ترتيب الكتاب. يمكن إضافة embeddings لاحقاً عند الحاجة، دون تغيير هوية المقاطع أو سجل الاستماع.
- يتضمن `story_segment_listening` و`story_playback_progress` حقولاً تسمح بإضافة sentence-level timing لاحقاً، لكن الإصدار الأول لا يدعي معرفة الجمل التي سُمعت.

### 5. اللغات والاستئناف

- لكل مقطع صف ترجمة/سيناريو مستقل لكل لغة من `ar`, `pl`, `en`، وصوت محفوظ للغة نفسها فقط عند توليده؛ لا تنشأ 3 نسخ تلقائياً لكل كتاب.
- العربية تعني العامية المصرية في السرد والحوار. البولندية يجب أن تستخدم قواعد النطق والحروف البولندية. الإنجليزية إنجليزية طبيعية مناسبة لطفلة.
- عند العودة باللغة نفسها: يستأنف نفس `segment_id` من `position_ms` المخزن، وتُحدّث القيمة دورياً وأثناء الإيقاف/الإغلاق.
- عند تغيير اللغة: يبقى `segment_id` كما هو، وتصبح نقطة البداية 0ms من صوت اللغة الجديدة؛ لا ننقل التوقيت لأن مدة التسجيل تختلف بين اللغات.
- عند عدم وجود صوت للغة المختارة، تنشأ وظيفة إعداد لهذه اللغة فقط. لا يُعرض رابط التخزين الدائم في قاعدة البيانات ولا signed URL محفوظ.

### 6. OCR Editing بلا Versioning

- لا جدول نسخ تاريخية ولا سجل نسخ قديمة. يضاف `story_source_revision` كرقم fencing فقط، وليس لحفظ نسخة من النص.
- تعديل OCR/الدرس يزيد الرقم وينشئ وظيفة جديدة. كل نتيجة تحمل revision وبصمة المصدر؛ قبل النشر يتحقق العامل أن revision ما زال الأحدث، وإلا يتخلص من الناتج القديم ولا يسمح له أن يستبدل الجديد.
- يحتفظ العامل بهوية UUID للمقاطع التي لم يتغير مصدرها عبر مطابقة بصمات النص والسياق المجاور. المقاطع المتأثرة فقط، مع الجيران اللازمين لاستقرار حدود المشاهد، تعاد معالجتها. المقاطع المتغيرة تستبدل دون الاحتفاظ بتاريخ إصدارات.
- بعد نجاح الاستبدال، تنظف ملفات الصوت القديمة للمقاطع التي لم تعد نشطة. أي تنظيف يتبع قاعدة البيانات ولا يترك روابط عامة.

### 7. التخزين والخصوصية

- Bucket باسم `story-audio` خاص، واسم المسار منظم مثل `user_id/book_id/segment_uuid/language.ext`.
- تخزن قاعدة البيانات `audio_path` وMIME/duration، ولا تخزن Signed URLs. الخادم ينشئ رابطاً قصير العمر للمقطع الحالي والتالي فقط.
- صلاحيات القراءة/الرفع/التعديل/الحذف محصورة بمجلد المستخدم؛ العامل يستخدم صلاحية الخادم ولا يكشف service-role key للمتصفح.
- نصوص OCR والقصص والصوت بيانات خاصة بالطفلة. لا تستخدم هذه البيانات خارج وظيفة السرد المطلوبة ولا تسجل نص الكتاب كاملاً في سجلات الأخطاء.

## مسؤوليات الجداول والعلاقات

| الجدول | المسؤولية والعلاقات |
|---|---|
| `pages` الموجود | مصدر OCR/الدرس، مع `story_source_revision` fencing counter مضاف؛ تابع لكتاب ومستخدم. |
| `story_segments` جديد | مقاطع المصدر المرتبة مع UUID ثابت وبصمات وحقائق/أحداث؛ يتبع صفحة وكتاباً ومستخدماً. |
| `story_segment_localizations` جديد | نص السرد والصوت وحالة التجهيز لكل مقطع ولغة؛ مفتاح مركب (segment, language). |
| `story_jobs` جديد | طابور دائم، حالات ومحاولات وموعد تشغيل وlease وtoken fencing؛ يتبع صفحة/كتاب/مستخدم. |
| `story_playback_progress` جديد | موضع الاستماع الحالي لكل مستخدم/كتاب/لغة، بالمللي ثانية وsegment UUID. |
| `story_segment_listening` جديد | يثبت أن المستخدم أكمل الاستماع لمقطع؛ هذا وحده يدخل RAG. |
| `storage.buckets/objects` | bucket خاص لملفات الصوت، مرتب بالمستخدم/الكتاب/المقطع/اللغة؛ الرابط المؤقت لا يخزن. |

الفهارس الأساسية: jobs بحسب الحالة وموعد التشغيل والـlease، segments بحسب الكتاب/الصفحة والترتيب، unique على `(page_id, stable_key)`، localizations بحسب `(segment_id, language)`, progress بحسب `(user_id, book_id)`, وسجل الإكمال بحسب `(user_id, segment_id)`.

## خطة التنفيذ بعد اعتماد الـMigration

1. **مراجعة البيئة الحية:** مقارنة الجداول والسياسات والإضافات والـStorage بين الإنتاج وملفات migrations. التحقق هل المشروع الحالي Lovable Cloud أو Supabase خارجي فعلياً، وهل `pg_cron`, `pg_net`, وVault متاحة. لا تشغيل قبل هذه المقارنة.
2. **تطبيق Migration معتمد:** بعد مراجعتك فقط، إضافة الأعمدة والجداول والسياسات والدوال الإضافية. لا Reset ولا حذف جداول/بيانات موجودة.
3. **اختبار TTS قبل التخزين:** تحديث عقد الدالة ليعيد MIME/container موثوقاً، وتطبيع/رفض أي خرج غير معروف. اختبار تسجيل قصير بالعربية المصرية والبولندية والإنجليزية عبر المسارين المتاحين، والتحقق من تشغيل الملفات مباشرة بعد رفعها إلى bucket خاص.
4. **العامل الدائم:** بناء الوظائف القصيرة، atomic claim، lease renewal، retry/backoff، حد محاولات، fencing بالـrevision، تنظيف العمل القديم، وجدولة Supabase أو Cloudflare واحدة فقط.
5. **إعداد الصفحة والمقاطع:** أتمتة تقسيم الصفحات، استخراج الأحداث والشخصيات، ثبات UUID، إنشاء النصوص المحلية عند الطلب، وتوليد الصوت وحفظ metadata.
6. **مشغل الكتاب:** تشغيل تسلسلي مع preload للمقطع التالي فقط، Signed URLs مؤقتة، pause/resume، حفظ موضع الاستماع، استمرار اللغة نفسها، وإعادة بداية المقطع عند تغيير اللغة.
7. **التفاعل وMini-RAG:** أسئلة بلغة الواجهة، استرجاع من المقاطع المكتملة فقط، عدم حرق أحداث، رسالة آمنة عند السؤال عن الجاري، ثم متابعة السرد.
8. **إعادة معالجة OCR:** اختبار تعديل كلمة/فقرة وحدود مقطع، عدم نشر job قديم، عدم حذف نتائج أحدث، وتجديد المقاطع المتأثرة فقط.
9. **الاختبارات والقبول:** انقطاع المتصفح بعد enqueue، انقطاع العامل بعد claim، انتهاء lease، retry بعد 503، فشل TTS، كتاب 300 صفحة، تبديل اللغة، signed URL منتهٍ، حذف صفحة/كتاب، RLS بين مستخدمين، سؤال عن مقطع غير مكتمل، والتحقق أن السرد لا يخترع أسماء أو أحداثاً. كذلك قياس زمن/تكلفة التجهيز لكل صفحة ولغة.

## شروط القبول

- أغلقت الطفلة المتصفح بعد إضافة الصفحة: يظل job في قاعدة البيانات ويُعالج لاحقاً.
- توقف العامل بعد claim: يعاد العمل بعد انتهاء lease دون أن يسمح لناتج قديم بالكتابة فوق نتيجة أحدث.
- تشغيل الكتاب لا يرسل 300 صفحة في طلب واحد، ويعرض الترتيب والتقدم بوضوح.
- استئناف اللغة نفسها دقيق بالمللي ثانية؛ تغيير اللغة يحافظ على segment UUID ويبدأ صوته من 0ms.
- Mini-RAG لا يسترجع مقاطع لم تكتمل، ولا يقدم spoilers.
- كل الملفات خاصة، Signed URLs قصيرة ولا تخزن.
- نجاح TTS مثبت فعلياً للغات الثلاث وبصيغة يمكن تشغيلها من المتصفح، لا مستنتج من وجود دالة فقط.

## SQL Migration مقترح للمراجعة فقط

> هذه مسودة additive مبنية على schema المشروع، ولم تُشغّل. يلزم أولاً التأكد من الـschema الحي ومن صلاحيات Extensions/Storage. لا تضف بيانات سرية لهذا الملف. الـCron schedule نفسه لا يُنشأ هنا لأن endpoint ووسيلة المصادقة يجب ضبطهما بأسرار البيئة بعد اختيار المستهلك.

```sql
begin;

-- Fencing counter only; does not store prior OCR versions.
alter table public.pages
  add column if not exists story_source_revision integer not null default 1;

create table if not exists public.story_segments (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  page_id uuid not null references public.pages(id) on delete cascade,
  page_number integer not null,
  segment_order integer not null check (segment_order >= 0),
  stable_key text not null,
  source_revision integer not null,
  source_hash text not null,
  source_text text not null,
  facts jsonb not null default '{}'::jsonb,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (page_id, stable_key)
);
create index if not exists story_segments_book_order_idx
  on public.story_segments (book_id, page_number, segment_order) where active;
create index if not exists story_segments_page_idx
  on public.story_segments (page_id, segment_order) where active;

create table if not exists public.story_segment_localizations (
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
  status text not null default 'queued'
    check (status in ('queued','processing','ready','failed')),
  narration_text text,
  audio_path text,
  audio_mime_type text,
  audio_duration_ms integer check (audio_duration_ms is null or audio_duration_ms >= 0),
  source_hash text not null,
  error_code text,
  updated_at timestamptz not null default now(),
  primary key (segment_id, language)
);
create index if not exists story_localizations_status_idx
  on public.story_segment_localizations (status, updated_at);

create table if not exists public.story_jobs (
  id uuid primary key default gen_random_uuid(),
  job_key text not null unique,
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  page_id uuid references public.pages(id) on delete cascade,
  segment_id uuid references public.story_segments(id) on delete cascade,
  job_type text not null check (job_type in ('process_page','prepare_segment','cleanup_audio')),
  language text check (language is null or language in ('ar','pl','en')),
  source_revision integer,
  status text not null default 'queued'
    check (status in ('queued','processing','retry','succeeded','failed','cancelled')),
  attempts integer not null default 0 check (attempts >= 0),
  max_attempts integer not null default 6 check (max_attempts between 1 and 12),
  available_at timestamptz not null default now(),
  lease_token uuid,
  lease_until timestamptz,
  heartbeat_at timestamptz,
  last_error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check ((status = 'processing') = (lease_token is not null and lease_until is not null))
);
create index if not exists story_jobs_claim_idx
  on public.story_jobs (available_at, created_at)
  where status in ('queued','retry');
create index if not exists story_jobs_expired_lease_idx
  on public.story_jobs (lease_until)
  where status = 'processing';
create index if not exists story_jobs_user_history_idx
  on public.story_jobs (user_id, created_at desc);

create table if not exists public.story_playback_progress (
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
  segment_id uuid references public.story_segments(id) on delete set null,
  position_ms bigint not null default 0 check (position_ms >= 0),
  updated_at timestamptz not null default now(),
  primary key (user_id, book_id)
);

create table if not exists public.story_segment_listening (
  user_id uuid not null,
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  completed_at timestamptz not null default now(),
  listened_ms bigint not null default 0 check (listened_ms >= 0),
  -- Reserved for future sentence-level timing; v1 uses segment completion only.
  timing_metadata jsonb not null default '{}'::jsonb,
  primary key (user_id, segment_id)
);
create index if not exists story_listening_user_completed_idx
  on public.story_segment_listening (user_id, completed_at desc);

-- Source changes increment a fencing counter, not a history/version table.
create or replace function public.bump_story_source_revision()
returns trigger language plpgsql set search_path = public, pg_temp as $$
begin
  if row(new.ocr_text, new.lesson) is distinct from row(old.ocr_text, old.lesson) then
    new.story_source_revision := old.story_source_revision + 1;
  end if;
  return new;
end;
$$;
drop trigger if exists pages_bump_story_source_revision on public.pages;
create trigger pages_bump_story_source_revision
before update of ocr_text, lesson on public.pages
for each row execute function public.bump_story_source_revision();

-- A ready page automatically enters the durable database-backed job table.
create or replace function public.enqueue_ready_story_page()
returns trigger language plpgsql security definer
set search_path = public, pg_temp as $$
begin
  if new.status = 'ready' and coalesce(new.ocr_text, '') <> '' then
    insert into public.story_jobs
      (job_key, user_id, book_id, page_id, job_type, source_revision)
    values
      ('page:' || new.id::text || ':rev:' || new.story_source_revision::text || ':process',
       new.user_id, new.book_id, new.id, 'process_page', new.story_source_revision)
    on conflict (job_key) do nothing;
  end if;
  return new;
end;
$$;
drop trigger if exists pages_enqueue_story_job on public.pages;
create trigger pages_enqueue_story_job
after insert or update of status, ocr_text, lesson on public.pages
for each row execute function public.enqueue_ready_story_page();

-- Atomic claim. Worker calls this with its service-role JWT. Expired leases are reclaimable.
create or replace function public.claim_story_jobs(p_limit integer default 3, p_lease_seconds integer default 300)
returns setof public.story_jobs
language plpgsql security definer
set search_path = public, pg_temp as $$
declare
  v_role text := auth.role();
begin
  if v_role is distinct from 'service_role' then
    raise exception 'service_role required';
  end if;

  update public.story_jobs
     set status = 'failed', lease_token = null, lease_until = null,
         last_error = coalesce(last_error, 'lease_expired_max_attempts'), updated_at = now()
   where status = 'processing' and lease_until < now() and attempts >= max_attempts;

  return query
  with candidates as (
    select id
      from public.story_jobs
     where attempts < max_attempts
       and available_at <= now()
       and (status in ('queued','retry') or (status = 'processing' and lease_until < now()))
     order by available_at, created_at
     for update skip locked
     limit greatest(1, least(coalesce(p_limit, 3), 10))
  ), claimed as (
    update public.story_jobs j
       set status = 'processing', attempts = j.attempts + 1,
           lease_token = gen_random_uuid(),
           lease_until = now() + make_interval(secs => greatest(60, least(coalesce(p_lease_seconds,300), 900))),
           heartbeat_at = now(), updated_at = now()
      from candidates c
     where j.id = c.id
     returning j.*
  ) select * from claimed;
end;
$$;

create or replace function public.renew_story_job_lease(p_job_id uuid, p_lease_token uuid, p_lease_seconds integer default 300)
returns boolean language plpgsql security definer
set search_path = public, pg_temp as $$
declare v_count integer;
begin
  if auth.role() is distinct from 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs
     set lease_until = now() + make_interval(secs => greatest(60, least(coalesce(p_lease_seconds,300), 900))),
         heartbeat_at = now(), updated_at = now()
   where id = p_job_id and status = 'processing' and lease_token = p_lease_token;
  get diagnostics v_count = row_count;
  return v_count = 1;
end;
$$;

create or replace function public.finish_story_job(
  p_job_id uuid, p_lease_token uuid, p_succeeded boolean,
  p_error text default null, p_retry_after_seconds integer default 60
)
returns boolean language plpgsql security definer
set search_path = public, pg_temp as $$
declare v_count integer;
begin
  if auth.role() is distinct from 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs
     set status = case
           when p_succeeded then 'succeeded'
           when attempts >= max_attempts then 'failed'
           else 'retry' end,
         available_at = case when not p_succeeded and attempts < max_attempts
           then now() + make_interval(secs => greatest(5, least(coalesce(p_retry_after_seconds,60), 3600)))
           else available_at end,
         last_error = case when p_succeeded then null else left(coalesce(p_error,'unknown_error'),1000) end,
         lease_token = null, lease_until = null, updated_at = now()
   where id = p_job_id and status = 'processing' and lease_token = p_lease_token;
  get diagnostics v_count = row_count;
  return v_count = 1;
end;
$$;

-- Private audio bucket. The worker uploads via service role; authenticated users can only access their own folder.
insert into storage.buckets (id, name, public)
values ('story-audio', 'story-audio', false)
on conflict (id) do update set public = false;

drop policy if exists story_audio_read_own on storage.objects;
create policy story_audio_read_own on storage.objects for select to authenticated
using (bucket_id = 'story-audio' and (storage.foldername(name))[1] = auth.uid()::text);
drop policy if exists story_audio_insert_own on storage.objects;
create policy story_audio_insert_own on storage.objects for insert to authenticated
with check (bucket_id = 'story-audio' and (storage.foldername(name))[1] = auth.uid()::text);
drop policy if exists story_audio_update_own on storage.objects;
create policy story_audio_update_own on storage.objects for update to authenticated
using (bucket_id = 'story-audio' and (storage.foldername(name))[1] = auth.uid()::text)
with check (bucket_id = 'story-audio' and (storage.foldername(name))[1] = auth.uid()::text);
drop policy if exists story_audio_delete_own on storage.objects;
create policy story_audio_delete_own on storage.objects for delete to authenticated
using (bucket_id = 'story-audio' and (storage.foldername(name))[1] = auth.uid()::text);

-- RLS: clients may read only their records; jobs and generated content are written by trusted server functions.
alter table public.story_segments enable row level security;
alter table public.story_segment_localizations enable row level security;
alter table public.story_jobs enable row level security;
alter table public.story_playback_progress enable row level security;
alter table public.story_segment_listening enable row level security;

grant select on public.story_segments, public.story_segment_localizations, public.story_jobs,
  public.story_playback_progress, public.story_segment_listening to authenticated;
grant all on public.story_segments, public.story_segment_localizations, public.story_jobs,
  public.story_playback_progress, public.story_segment_listening to service_role;

drop policy if exists story_segments_read_own on public.story_segments;
create policy story_segments_read_own on public.story_segments for select to authenticated
using (user_id = auth.uid());
drop policy if exists story_localizations_read_own on public.story_segment_localizations;
create policy story_localizations_read_own on public.story_segment_localizations for select to authenticated
using (exists (select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()));
drop policy if exists story_jobs_read_own on public.story_jobs;
create policy story_jobs_read_own on public.story_jobs for select to authenticated
using (user_id = auth.uid());
drop policy if exists story_progress_own on public.story_playback_progress;
create policy story_progress_own on public.story_playback_progress for all to authenticated
using (user_id = auth.uid() and exists (select 1 from public.books b where b.id = book_id and b.user_id = auth.uid()))
with check (user_id = auth.uid() and exists (select 1 from public.books b where b.id = book_id and b.user_id = auth.uid()));
drop policy if exists story_listening_own on public.story_segment_listening;
create policy story_listening_own on public.story_segment_listening for all to authenticated
using (user_id = auth.uid() and exists (select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()))
with check (user_id = auth.uid() and exists (select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()));

revoke all on function public.claim_story_jobs(integer, integer) from public, anon, authenticated;
revoke all on function public.renew_story_job_lease(uuid, uuid, integer) from public, anon, authenticated;
revoke all on function public.finish_story_job(uuid, uuid, boolean, text, integer) from public, anon, authenticated;
grant execute on function public.claim_story_jobs(integer, integer) to service_role;
grant execute on function public.renew_story_job_lease(uuid, uuid, integer) to service_role;
grant execute on function public.finish_story_job(uuid, uuid, boolean, text, integer) to service_role;

commit;
```

## ما لا يشمله SQL أعلاه

- لا ينشئ Cron تلقائياً ولا يخزن Service Role key أو أي سر. بعد اعتماد الـMigration نحدد المستهلك، نضيف secret من لوحة Supabase/Vault أو إعدادات Cloudflare، ثم نضبط cron ليستدعي Edge Function كل دقيقة. الوظيفة نفسها تتحقق من secret، تطالب بعدد محدود من الأعمال، وتجدد الـlease عند الحاجة.
- لا ينشئ embeddings أو سجل نسخ OCR؛ الأول يمكن إضافته لاحقاً، والثاني مستبعد عمداً.
- لا يخزن Signed URLs أو ينتج ملفات TTS؛ التنفيذ يتضمن دالة server-side لذلك، مع التحقق من MIME والصوت قبل الرفع.
- `story_source_revision` وقيود RLS في المسودة يجب مقارنتها بالسياسات والـschema الحي قبل تشغيلها. بما أن قراءة قاعدة الإنتاج لم تنجح في هذه المراجعة، فلا أنصح بتنفيذ هذا النص مباشرة على Production قبل الفحص.

## المراجع الفنية

- [Supabase PGMQ](https://supabase.com/docs/guides/queues/pgmq)
- [Supabase Edge Function limits](https://supabase.com/docs/guides/functions/limits)
- [Cloudflare Workers limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Supabase Storage access model](https://supabase.com/docs/guides/storage/buckets/fundamentals)
- [Gemini TTS model documentation](https://ai.google.dev/gemini-api/docs/models/gemini-2.5-flash-preview-tts)
