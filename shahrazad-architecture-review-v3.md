# مراجعة معمارية وتصميم نظام "احكي يا شهرزاد" (Shahrazad Architecture Review — V3)

- **تاريخ التحديث:** نوفمبر 2026
- **الحالة:** وثيقة معمارية ومسودة SQL Migration مرجعية **للمراجعة فقط (Review Only — Do Not Execute)**
- **تنبيه أمان صارم:** لا تشغّل هذا النص على قاعدة الإنتاج، ولا تعدّل الكود البرمجي الحالي للمشروع أو تحذف بياناته. قاعدة الإنتاج الحية لم يتم قراءتها مباشرة من البيئة؛ لذلك يجب تصدير Schema الإنتاج ومقارنتها وإجراء اختبارات Staging قبل اعتماد أي تغيير.

---

## 1. ملخص التعديلات الجوهرية في الإصدار الثالث (V3 Changelog)

بناءً على المراجعة الدقيقة لملاحظات V2، يعالج هذا الإصدار النقاط التقنية العشر بالكامل دون إدخال تعقيدات غير مطلوبة (مثل Vector DB، أو Redis، أو أنظمة Versioning تاريخية معقدة):

1. **إبطال المخرجات القديمة الصارم (Strict Stale Invalidation):**
   - في دالة استبدال المشاهد `story_replace_segment_range`، عند تغيّر `source_hash` للمقطع يتم إبطال حالات السيناريو والصوت في `story_segment_localizations` فوراً (`script_status = 'failed'`, `audio_status = 'failed'`) ومسح مسارات الصوت ومطابقة الـhashes القديمة لمنع تقديم محتوى قديم.
   - في `story_start_playback` و`story_get_audio_for_signing`، يُشترط صراحة تطابق `l.audio_source_hash = s.source_hash` وكون المقطع فعالاً `s.active`، لمنع تشغيل أو توقيع أي مقطع تغيرت صفحاته.
   - في Mini-RAG `story_search_completed_segments`، يتم قصر البحث على المقاطع الفعالة المكتملة التي يطابق محتواها مصدر المقطع الحالي.

2. **استبدال نطاق المشاهد الذري المنضبط (Guarded Atomic Range Replacement):**
   - حماية قيد `(book_id, stable_key)`: التحقق الصارم من أن أي Segment في الحمولة الجديدة إما غير موجود مسبقاً أو ينتمي بالفعل إلى المشاهد المتأثرة بالنطاق الحالي (`v_old_ids`). يُرفض الطلب فوراً إذا حاول تعديل Segment سليم خارج النطاق المستهدف.
   - التحقق من عدم تقاطع المشاهد مع حدود النطاق؛ إذا كان هناك مشهد يمتد جزئياً خارج نطاق الصفحات المطلوبة، يرفض الاستبدال ويطلب من العامل توسيع نافذة الصفحات لضمان سلامة السياق المشهدي.

3. **دمج طلبات الصفحات المتداخلة (Job Coalescing):**
   - معالجة مشكلة توليد مئات المهام لكتاب كبير (300 صفحة): تم تزويد آلية إدراج المهام بالدمج الذكي (Coalescing). عند إدراج نطاق جديد يتداخل مع نطاق في حالة `queued` لنفس الكتاب، يتم تحديث النطاق القائم (`scope_start_page`, `scope_end_page`) ليحتوي التغيير المشترك بدلاً من مضاعفة المهام.
   - جعل دالة طلب التجهيز المباشر `story_request_preparation` عملية Idempotent تعتمد مفتاحاً حتمياً لكل (مستخدم، كتاب، لغة، نطاق) وتكتفي برفع الأولوية إذا كانت المهمة قيد الانتظار أو إعادة المحاولة.
   - صياغة خطة Backfill تعتمد نوافذ معالجة منتظمة غير متداخلة (Windows of 10-15 pages) لتغطية الكتب الكبيرة بعدد محدود ومضبوط من المهام.

4. **معالجة حذف الصفحات وتغيير حالتها وتنظيف التخزين (Page Deletion & Storage Cleanup):**
   - إضافة Trigger على جدول `pages` عند الحذف (`ON DELETE`) أو تغيير الحالة من `ready` إلى غير متاحة، يقوم بإبطال المشاهد المعتمدة على الصفحة وتوليد مهمة معالجة لنطاق الصفحات المحيط.
   - إضافة آلية تنظيف التخزين عند حذف الكتب: نظراً لأن `ON DELETE CASCADE` في قاعدة البيانات لا يحذف ملفات Supabase Storage تلقائياً، يتم تسجيل مهمة تنظيف (`cleanup_book_storage`) في جدول المهام ليقوم العامل عبر `service_role` بحذف مجلد الكتاب بالكامل من Bucket `story-audio`.

5. **إعادة الترتيب الجماعي للصفحات في عملية ذرية واحدة (Batch Page Reordering):**
   - توفير دالة RPC ذرية `story_reorder_pages` تستقبل مصفوفة الترتيب الجديد بالكامل في عملية Transaction واحدة.
   - تقوم الدالة بحساب النطاق المتأثر الإجمالي (أدنى وأقصى رقم صفحة تغير موضعها)، وزيادة عداد التغيير `story_source_change_token` للصفحات المنقولة، وإدراج مهمة معالجة مدمجة واحدة للنطاق المتأثر مع جيرانه.

6. **RAG متعدد اللغات مستقل عن لغة الاستماع (Decoupled Multilingual RAG):**
   - فصل لغة الاستماع عن لغة إجابة شهرزاد: إذا استمعت الطفلة للجزء بالعربية وغيرت الواجهة إلى البولندية، لا يُشترط وجود الترجمة البولندية لذلك الجزء مسبقاً.
   - يستند البحث إلى جدول المقاطع المكتملة وحقائقها المعيارية وملخصاتها المتاحة، ثم يتولى نموذج اللغة (Gemini) صياغة الإجابة بلغة الواجهة الحالية مع التزامه الصارم بعدم حرق الأحداث اللاحقة.

7. **تدقيق FTS للبولندية والعامية المصرية دون Vector Database:**
   - الحفاظ على بساطة المعمارية عبر PostgreSQL FTS مع `simple` dictionary.
   - تعويض غياب الـ Stemming للغة البولندية المعقدة تصريفياً والعامية المصرية متعددة السوابق من خلال استخراج كيانات وأفعال رئيسية بصيغ متعددة ضمن حقل `facts` (JSONB) أثناء التلخيص، وإدراجها في فهرس الـ FTS.
   - تحديد سيناريوهات اختبار للكلمات المشتقة والمرادفات الشائعة لدى الأطفال.

8. **معمارية التعرف على الصوت (STT: Push-to-Talk Audio Buffering vs WebSockets):**
   - تحليل معماري شامل يقارن بين استخدام WebSocket دائم وبين تسجيل السؤال (Push-to-Talk) وإرساله كملف صوتي قصير عبر REST/Server Function.
   - ترجيح أسلوب التسجيل المباشر للسؤال لكونه أقل تكلفة (صفر تكلفة خاملة أثناء الاستماع للقصة)، وأكثر استقراراً على Safari iOS وأجهزة الموبايل، ويوفر سياقاً صوتياً كاملاً يرفع دقة التعرف على اللهجة المصرية والبولندية.

9. **ثبات نبرة وشخصية الصوت (Voice Profile Consistency):**
   - اعتماد بطاقة توصيف صوتي ثابتة (`VoiceProfile`) لكل لغة مع تحديد اسم الصوت المعياري، إعدادات النبرة والسرعة، وتعليمات الأسلوب لضمان عدم تذبذب شخصية شهرزاد بين المقاطع.

10. **مجموعة اختبارات القبول الموسعة (Expanded Acceptance Suite):**
    - إضافة جميع السيناريوهات الحرجة وحالات التسابق (Race Conditions) المحددة من قبل المستخدم لاختبارها قبل النشر على الإنتاج.

---

## 2. المعمارية العامة ومسؤوليات المكونات

النظام مبني على مبدأ **التجهيز الخلفي المسبق (Durable Background Preparation)** لتجنب أي معالجة ثقيلة وقت الاستماع المباشر للطفلة:

```
[إضافة/تعديل/إعادة ترتيب صفحات]
         │
         ▼
[PostgreSQL Triggers / RPCs] ──(Coalescing & Fencing)──► [story_jobs Queue]
                                                               │
                                                               ▼
[Server Function / Worker API] ◄──(Claim via SKIP LOCKED)──────┘
         │
         ├── 1. تقطيع المشاهد الدلالية (Semantic Segmentation عبر عدة صفحات)
         ├── 2. توليد سيناريو شهرزاد المخصص (Script Generation باللغة المطلوبة)
         ├── 3. توليد الصوت عبر Gemini TTS المعياري وفق Voice Profile
         ├── 4. رفع ملف الصوت إلى Private Supabase Bucket (`story-audio`)
         │
         ▼
[Atomic Publish RPC] ──(التحقق من صحة Lease ومطابقة Source Tokens)──► [story_segments & Localizations]
```

### القواعد المعمارية الثابتة:
1. **لا سيرفرات مستقلة ولا Redis:** طابور المهام مدعوم كلياً بجدول `story_jobs` وفهارس جزئية مع `FOR UPDATE SKIP LOCKED`.
2. **لا نظام Version History:** نعتمد فقط على عداد تزامن بسيط `story_source_change_token` كـ Concurrency Guard لمنع نشر نتائج قديمة. عند تغيير المصدر يُستبدل القديم ويلغى.
3. **أمان تام لملفات الصوت والتخزين:** الـ Bucket خاص تماماً، وتصدر روابط Signed URLs قصيرة الأجل (5-10 دقائق) عبر Server Function بعد التحقق من ملكية ولي الأمر للكتاب. المتصفح لا يمتلك أي صلاحية تعديل أو رفع أو حذف في الـ Storage.
4. **عزل هوية الطفل:** لا توجد هوية Auth منفصلة للطفل؛ كل السجلات والملفات تتبع حساب ولي الأمر `auth.uid()`.

---

## 3. نموذج البيانات التفصيلي (Data Model)

| الجدول | الدور والمسؤولية | المفاتيح والفهارس |
|---|---|---|
| `pages` (الموجود) | مصدر النص والصور والدروس. يضاف له عداد `story_source_change_token` لمنع تداخل التعديلات. | PK: `id`. Index: `(book_id, page_number)`. |
| `story_segments` (جديد) | المشهد الدلالي على مستوى الكتاب (قد يغطي صفحة واحدة أو عدة صفحات متتابعة). يحتوي النص المصلي والملخص والحقائق. | PK: `id`. Unique: `(book_id, stable_key)`. Index: `(book_id, segment_order)`. |
| `story_segment_pages` (جديد) | يمثل علاقة Many-to-Many بين المقطع والصفحات التي يقتبس منها، مع ترتيب الصفحة داخل المشهد ولقطة لعداد التغيير. | PK: `(segment_id, page_id)`. Index: `page_id`. |
| `story_segment_localizations` (جديد) | يحتوي الترجمة والسيناريو وملف الصوت لكل مقطع ولغة (`ar, pl, en`). يفصل تماماً بين حالة السيناريو وحالة الصوت. | PK: `(segment_id, language)`. Indexes على حالات `script_status` و`audio_status`. |
| `story_jobs` (جديد) | طابور المهام الدائم؛ يدعم التأجير (Leasing)، الأولويات، المحاولات، والدمج الذكي للنطاقات المتداخلة. | PK: `id`. Unique: `job_key`. Indexes للـ Claim السريع والانتهاء. |
| `story_playback_progress` (جديد) | يحفظ موضع الاستماع بالمللي ثانية لكل مستخدم وكتاب ولغة بشكل مستقل تماماً. | PK: `(user_id, book_id, language)`. |
| `story_playback_sessions` (جديد) | جلسات الاستماع المؤقتة للتحقق من الاستماع الفعلي ومنع القفز الوهمي لنهاية المقطع. | PK: `id`. Index: `(user_id, book_id)`. |
| `story_segment_listening` (جديد) | السجل المعتمد لإكمال الاستماع؛ هو المصدر الأساسي الحصري لمحرك Mini-RAG لمنع حرق الأحداث. | PK: `(user_id, segment_id, language)`. |

---

## 4. إبطال المحتوى القديم (Stale Content Invalidation) والـ Fencing

### آليات الحماية الفورية:
1. **عند تعديل OCR أو نص صفحة أو رقمها:**
   - يزداد `story_source_change_token` الخاص بالصفحة بمقدار 1.
   - تصبح لقطات `source_change_token_snapshot` في جدول `story_segment_pages` غير متطابقة.
   - أي محاولة من عامل خلفي لنشر نتيجة استندت إلى اللقطة القديمة تفشل ذرياً وترفض عبر `story_publish_segment_stage`.

2. **عند تحديث المقطع الدلالي `story_segments`:**
   - إذا تغير `source_hash` الخاص بالمقطع، يتم فوراً في نفس المعاملة (Transaction) تحديث جميع سجلات `story_segment_localizations` التابعة له:
     - تعيين `script_status = 'failed'`, `audio_status = 'failed'` مع مسح `script_source_hash` و`audio_source_hash`.
     - تسجيل مسارات الصوت القديمة في طابور مهام التنظيف ليتم حذفها من الـ Bucket.

3. **حماية مسار الاستماع وتوقيع الروابط:**
   - في `story_start_playback`: يتم رفض بدء الجلسة إذا كان `audio_status <> 'ready'` أو `audio_source_hash <> story_segments.source_hash`.
   - في `story_get_audio_for_signing`: تفحص الدالة تطابق الـ Hash مع المصدر الحي. إذا حدث تعديل للصفحة أثناء تشغيل المقطع، يمنع توقيع روابط جديدة للملف القديم.

---

## 5. دمج المهام (Job Coalescing) واستراتيجية الكتب الكبيرة

### مشكلة التكرار والحل:
عند رفع عدة صفحات متعاقبة أو إجراء Backfill لكتاب مكون من 300 صفحة، فإن تطبيق نطاق `[page-1, page+1]` لكل صفحة ينتج 300 مهمة متداخلة تتنافس على نفس الأقفال.

### الحل المعتمد في V3:
1. **Trigger Coalescing:**
   عند إضافة أو تعديل صفحة، يفحص الـ Trigger ما إذا كانت هناك مهمة `queued` لنفس الكتاب والنوع `process_range`. إذا وجد مهمة يتداخل نطاقها مع النطاق الجديد (`scope_start_page <= new_end + 1 AND scope_end_page >= new_start - 1`)، يقوم بتوسيع نطاق المهمة القائمة بدلاً من إدراج مهمة جديدة:
   ```sql
   UPDATE public.story_jobs
      SET scope_start_page = LEAST(scope_start_page, v_start),
          scope_end_page   = GREATEST(scope_end_page, v_end),
          updated_at       = now()
    WHERE id = existing_job_id;
   ```

2. **Idempotent Playback Request:**
   دالة طلب الاستماع الفوري `story_request_preparation` تعتمد مفتاحاً محدداً:
   `'request:' || p_book_id || ':' || p_language || ':p' || v_page.page_number`
   إذا كان الطلب موجوداً في حالة `queued` أو `retry`، تكتفي برفع أولويته (`priority = 0`) دون تكرار الصفوف.

3. **خطة الـ Backfill المقسم لنوافذ (Windowed Backfill):**
   بدلاً من معالجة كل صفحة كـ Job مستقل، يقسم الـ Backfill الكتاب الكبير إلى نوافذ منتظمة من 10 إلى 15 صفحة (مع تداخل صفحة واحدة لضمان تماسك المشاهد عبر الحدود). لكتاب من 300 صفحة، ينتج ذلك حوالي 20 إلى 25 مهمة فقط، تنفذ بأولوية منتظمة.

---

## 6. حذف الصفحات وتغيير ترتيبها وتنظيف التخزين

### حذف أو تعطيل صفحة:
- تم ربط Trigger قبل الحذف `BEFORE DELETE` أو عند تغيير الحالة من `ready` على جدول `pages`.
- يقوم الـ Trigger بتعطيل المقاطع المتأثرة (`active = false`) وإبطال ترجماتها وصوتياتها، ثم يدرج مهمة لإعادة تجهيز نطاق الصفحات المحيط لملء الفراغ الحكائي.

### تنظيف ملفات الصوت عند حذف الكتاب:
- حذف الكتاب في قاعدة البيانات ينفذ `CASCADE` على جداول SQL فقط، مما يترك ملفات MP3/WAV مهملة في الـ Bucket.
- تم إضافة Trigger على جدول `books` عند الحذف (`BEFORE DELETE`) يدرج مهمة خاصة `cleanup_book_storage` تحتوي على مسار مجلد المالك والكتاب.
- يتولى العامل معالجة هذه المهمة عبر استدعاء Supabase Storage API بـ `service_role` ومسح المجلد بالكامل.

### إعادة الترتيب الجماعي (Batch Page Reordering):
- توفر دالة `story_reorder_pages` حلاً جذرياً لمشكلة إعادة الترتيب:
  1. تقبل مصفوفة JSON تحتوي الأزواج `[{"page_id": "...", "new_page_number": X}]`.
  2. تغلق صفحات الكتاب `FOR UPDATE` في transaction واحدة.
  3. تطبق الترقيم الجديد بأمان لمنع تعارض المفاتيح الفريدة.
  4. تحسب أدنى وأقصى رقم صفحة تأثر بالنقل وتزيد عدادات التغيير.
  5. تدرج **مهمة معالجة مدمجة واحدة** تغطي النطاق المتأثر الإجمالي مع صفحة سياق سابقة ولاحقة.

---

## 7. RAG متعدد اللغات وFTS دون Vector Database

### فصل لغة الاستماع عن لغة السؤال:
- **المشكلة:** إذا استمعت الطفلة للجزء بالعامية المصرية ثم حولت الواجهة إلى البولندية وسألت سؤالاً بالبولندية، فإن اشتراط وجود ترجمة بولندية مسبقة للجزء المستمع سيعطل الإجابة.
- **الحل:** 
  1. يحدد النظام المقاطع التي **أكملت الطفلة الاستماع إليها فعلياً** عبر جدول `story_segment_listening` (بغض النظر عن اللغة التي استمعت بها).
  2. يتم استرجاع الحقائق والملخصات المعيارية المخزنة في `story_segments` (والتي تحتوي الأسماء الأصلية، الأحداث، والشخصيات بالبولندية والعربية).
  3. يتم تزويد نموذج Gemini بالحقائق المكتملة وسؤال الطفلة، مع توجيهه الصريح: *"أجب بلغة الواجهة الحالية (${target_language}) بالاعتماد حصراً على هذه الحقائق المكتملة دون حرق أي أحداث قادمة"*.

### استراتيجية FTS مع اللغتين البولندية والعربية:
- استخدام قاموس `simple` يتجنب افتراضات القواميس الإنجليزية، لكنه لا يعالج التصريفات الصرفية البولندية (Inflections) أو سوابق العامية المصرية.
- **الحل العملي:** أثناء تلخيص المقطع في العامل الخلفي، يتم استخراج مصفوفة كيانات غنية (Entities & Keywords) وتخزينها في عمود `facts` بصيغ متعددة (مثل: اسم الشخصية بصيغة الرفع والمجرور، الكلمات الأساسية بالمصري والبولندي).
- يشمل فهرس `tsvector` المشتق حقل `facts` بالكامل، مما يضمن عثور محرك FTS على المشهد حتى لو استخدمت الطفلة صيغة صرفية مختلفة أو كلمة بالعامية.

---

## 8. معمارية التعرف على الصوت (STT): التسجيل المباشر مقابل WebSockets

| وجه المقارنة | Gemini Live WebSocket (محادثة مستمرة) | Push-to-Talk (تسجيل السؤال وإرساله دفعة واحدة) |
|---|---|---|
| **الاستقرار والتوافق** | هش على متصفحات الموبايل (خاصة Safari iOS) عند انقطاع الشبكة أو التبديل بين التطبيقات. | مستقر للغاية عبر معيار `MediaRecorder` المدمج في جميع المتصفحات. |
| **التكلفة** | تكلفة فتح جلسة WebSocket مستمرة واستهلاك Audio In/Out طوال وقت الحكاية حتى أثناء الصمت. | صفر تكلفة أثناء الاستماع؛ استهلاك الصوت محصور فقط في مدة سؤال الطفلة (3-5 ثوانٍ). |
| **زمن الاستجابة والتعقيد** | يحتاج إدارة معقدة لإشارات الميكروفون وإلغاء الصدى الصوتي أثناء تشغيل القصة. | يوقف تشغيل القصة مؤقتاً، يسجل السؤال، يرسله في طلب واحد، ويعيد الإجابة بسرعة فائقة. |
| **دقة التعرف على اللهجات** | يعتمد على المعالجة التتابعية اللحظية التي قد تخطئ في الكلمات الأولى. | يرسل الملف الصوتي كاملاً فيحصل الموديل على السياق الصوتي الكامل للعامية والبولندية. |
| **القرار المعماري** | غير موصى به في V1 كطبقة أسئلة. | **الخيار المعتمد لـ V1:** تسجيل صوت السؤال وإرساله إلى Server Function تعالج التفريغ والإجابة. |

---

## 9. اتساق الصوت وشخصية شهرزاد (Voice Profiles)

لضمان عدم تغير طبقة الصوت أو لكنة شهرزاد بين المقاطع الصوتية، تعتمد المعمارية توصيفاً صوتياً صارماً يمرر في كل استدعاء لموديل TTS:

```typescript
export const SHAHRAZAD_VOICE_PROFILES = {
  ar: {
    model: 'gemini-3.8-flash-tts',
    voiceName: 'Puck', // أو الصوت العربي المعتمد بعد الاختبار
    languageCode: 'ar-EG',
    speakingRate: 0.95,
    pitch: 0.0,
    stylePrompt: 'أنتِ شهرزاد، حكواتية مصرية دافئة ولطيفة تحكي قصة أطفال بأسلوب ممتع ومشوق، نطقك عامي مصري محبب وسلس، وتنطقين الأسماء والكلمات البولندية كما هي بوضوح ولطف دون تكلف.'
  },
  pl: {
    model: 'gemini-3.8-flash-tts',
    voiceName: 'Aoede', // الصوت المعتمد للنطق البولندي
    languageCode: 'pl-PL',
    speakingRate: 0.95,
    pitch: 0.0,
    stylePrompt: 'Jesteś Szeherezadą – ciepłą, melodyjną narratorką bajek dla dzieci. Czytasz wyraźnie, z naturalną dziecięcą intonacją, zachowując spokój i magię opowieści.'
  },
  en: {
    model: 'gemini-3.8-flash-tts',
    voiceName: 'Aoede',
    languageCode: 'en-US',
    speakingRate: 1.0,
    pitch: 0.0,
    stylePrompt: 'You are Shahrazad, a warm, imaginative and soothing storyteller for children. Speak clearly, engagingly, and with natural warmth.'
  }
};
```

---

## 10. اختبارات القبول الموسعة (Expanded Acceptance Suite)

يجب أن تجتاز المنظومة الاختبارات التالية على بيئة Staging قبل التفكير في نقلها للإنتاج:

1. **سباق تعديل OCR مع طلب التشغيل:** تعديل نص صفحة ثم طلب تشغيل المقطع فوراً -> النظام يرفض تشغيل الصوت القديم (`audio_source_hash` غير مطابق)، ويبدأ بتجهيز الصوت الجديد مع إشعار انتظار لطيف للطفلة.
2. **حذف صفحة مرتبطة بمشهد متعدد الصفحات:** حذف صفحة وسطى في مشهد يغطي 3 صفحات -> يتم إبطال المشهد تلقائياً، إلغاء ترجماته، وتوليد مهمة لإعادة صياغة المشاهد للصفحتين المتبقيتين.
3. **إعادة ترتيب 5 صفحات دفعة واحدة:** استدعاء `story_reorder_pages` لخمس صفحات -> يتم تحديث أرقام الصفحات في معاملة واحدة، وحساب النطاق المشترك، وإصدار **Job واحد مدمج** فقط بدلاً من 5 مهام منفصلة.
4. **دمج مهام Backfill لكتاب 300 صفحة:** تشغيل الـ Backfill المقسم لنوافذ -> التأكد من إنشاء نحو 25 مهمة منظمة فقط، مع عدم وجود مهام متطابقة أو تداخل تنافسي معطل للـ DB.
5. **تبديل لغة الاستماع والسؤال عن أحداث سابقة:** استماع الطفلة لـ 3 مقاطع بالعربية ثم تحويل الواجهة للبولندية وسؤالها عن اسم شخصية ظهرت في المقطع الثاني -> شهرزاد تسترجع الحقائق المكتملة وتجيب بالبولندية دون اشتراط وجود صوت بولندي للمقطع القديم.
6. **فشل TTS مع جاهزية السيناريو:** فشل خدمة توليد الصوت بعد اكتمال كتابة السيناريو بنجاح -> يبقى `script_status = 'ready'`، وتتحول `audio_status = 'retry'`؛ عند إعادة المحاولة يعاد طلب الصوت فقط دون إعادة كتابة النص.
7. **انهيار العامل بعد رفع الصوت وقبل الاعتماد الذري:** رفع ملف MP3 إلى التخزين ثم انقطاع العامل قبل استدعاء `story_publish_segment_stage` -> عملية الـ Publish تفشل لانتهاء الـ Lease، والملف المهمل يسجل ليتم حذفه عبر مهمة cleanup.
8. **حذف الكتاب وتنظيف التخزين الفعلي:** حذف كتاب يحتوي على 50 ملف صوتي -> تنفيذ الحذف لقواعد البيانات وإدراج مهمة `cleanup_book_storage`، والتأكد من مسح جميع الملفات من Supabase Storage فعلياً.

---

## 11. مسودة الـ SQL Migration المقترحة للمراجعة فقط (V3)

```sql
-- ============================================================================
-- SHAHRAZAD SYSTEM SCHEMA & RPCS (V3)
-- NOTICE: FOR ARCHITECTURAL REVIEW ONLY. DO NOT EXECUTE ON PRODUCTION.
-- Requires comparison with live production schema and staging verification.
-- ============================================================================

begin;

-- 1. Ensure Concurrency Guard on existing pages table
alter table public.pages
  add column if not exists story_source_change_token bigint not null default 1;

-- 2. Story Segments (Semantic Scenes spanning one or more pages)
create table if not exists public.story_segments (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  segment_order integer not null check (segment_order >= 0),
  stable_key text not null,
  source_hash text not null,
  source_text text not null,
  summary_text text not null default '',
  facts jsonb not null default '[]'::jsonb,
  search_document tsvector generated always as (
    to_tsvector('simple', coalesce(source_text, '') || ' ' || coalesce(summary_text, '') || ' ' || coalesce(facts::text, ''))
  ) stored,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (book_id, stable_key),
  constraint story_segments_book_order_uq unique (book_id, segment_order) deferrable initially deferred
);

create index if not exists story_segments_book_order_idx on public.story_segments(book_id, segment_order) where active;
create index if not exists story_segments_search_idx on public.story_segments using gin(search_document) where active;
create index if not exists story_segments_user_book_idx on public.story_segments(user_id, book_id);

-- 3. Many-to-Many Mapping Between Segments and Pages
create table if not exists public.story_segment_pages (
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  page_id uuid not null references public.pages(id) on delete cascade,
  source_order integer not null check (source_order >= 0),
  source_change_token_snapshot bigint not null,
  primary key (segment_id, page_id),
  unique (segment_id, source_order)
);

create index if not exists story_segment_pages_page_idx on public.story_segment_pages(page_id);
create index if not exists story_segment_pages_segment_idx on public.story_segment_pages(segment_id, source_order);

-- 4. Segment Localizations (Script and Audio separated)
create table if not exists public.story_segment_localizations (
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar', 'pl', 'en')),
  script_status text not null default 'queued'
    check (script_status in ('queued', 'processing', 'ready', 'retry', 'failed')),
  script_text text,
  localized_summary text not null default '',
  localized_facts jsonb not null default '[]'::jsonb,
  search_document tsvector generated always as (
    to_tsvector('simple', coalesce(script_text, '') || ' ' || coalesce(localized_summary, '') || ' ' || coalesce(localized_facts::text, ''))
  ) stored,
  script_source_hash text,
  script_error text,
  audio_status text not null default 'queued'
    check (audio_status in ('queued', 'processing', 'ready', 'retry', 'failed')),
  audio_path text,
  audio_mime_type text,
  audio_duration_ms integer check (audio_duration_ms is null or audio_duration_ms >= 0),
  audio_source_hash text,
  audio_error text,
  updated_at timestamptz not null default now(),
  primary key (segment_id, language)
);

create index if not exists story_localizations_search_idx on public.story_segment_localizations using gin(search_document);
create index if not exists story_localizations_audio_status_idx on public.story_segment_localizations(audio_status, updated_at);
create index if not exists story_localizations_script_status_idx on public.story_segment_localizations(script_status, updated_at);

-- 5. Durable Story Jobs Queue with Coalescing Support
create table if not exists public.story_jobs (
  id uuid primary key default gen_random_uuid(),
  job_key text not null unique,
  parent_job_id uuid references public.story_jobs(id) on delete set null,
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  page_id uuid references public.pages(id) on delete cascade,
  segment_id uuid references public.story_segments(id) on delete cascade,
  scope_start_page integer,
  scope_end_page integer,
  job_type text not null check (job_type in ('process_range', 'prepare_script', 'prepare_audio', 'cleanup_audio', 'cleanup_book_storage')),
  language text check (language is null or language in ('ar', 'pl', 'en')),
  source_change_token bigint,
  priority smallint not null default 30 check (priority between 0 and 100),
  status text not null default 'queued'
    check (status in ('queued', 'processing', 'retry', 'succeeded', 'failed', 'cancelled')),
  attempts integer not null default 0 check (attempts >= 0),
  max_attempts integer not null default 6 check (max_attempts between 1 and 12),
  available_at timestamptz not null default now(),
  lease_token uuid,
  lease_until timestamptz,
  heartbeat_at timestamptz,
  payload jsonb not null default '{}'::jsonb,
  last_error_code text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check ((status = 'processing') = (lease_token is not null and lease_until is not null))
);

create index if not exists story_jobs_claim_idx on public.story_jobs(priority, available_at, created_at)
  where status in ('queued', 'retry');
create index if not exists story_jobs_expired_idx on public.story_jobs(lease_until)
  where status = 'processing';
create index if not exists story_jobs_owner_idx on public.story_jobs(user_id, created_at desc);

-- 6. Playback Progress (Independent per language)
create table if not exists public.story_playback_progress (
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  language text not null check (language in ('ar', 'pl', 'en')),
  segment_id uuid references public.story_segments(id) on delete set null,
  position_ms bigint not null default 0 check (position_ms >= 0),
  updated_at timestamptz not null default now(),
  primary key (user_id, book_id, language)
);

-- 7. Playback Sessions (Monotonic validation against skipping)
create table if not exists public.story_playback_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar', 'pl', 'en')),
  audio_duration_ms integer not null check (audio_duration_ms > 0),
  started_position_ms bigint not null default 0,
  position_ms bigint not null default 0,
  max_position_ms bigint not null default 0,
  accumulated_play_ms bigint not null default 0,
  is_playing boolean not null default true,
  last_checkpoint_at timestamptz not null default now(),
  started_at timestamptz not null default now(),
  completed_at timestamptz,
  created_at timestamptz not null default now()
);

create index if not exists story_playback_sessions_owner_idx on public.story_playback_sessions(user_id, book_id, created_at desc);

-- 8. Validated Listening History (Source of truth for Mini-RAG)
create table if not exists public.story_segment_listening (
  user_id uuid not null,
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar', 'pl', 'en')),
  completed_at timestamptz not null default now(),
  listened_ms bigint not null check (listened_ms >= 0),
  timing_metadata jsonb not null default '{}'::jsonb,
  primary key (user_id, segment_id, language)
);

create index if not exists story_listening_completed_idx on public.story_segment_listening(user_id, language, completed_at desc);

-- ----------------------------------------------------------------------------
-- TRIGGERS & CONCURRENCY CONTROLS
-- ----------------------------------------------------------------------------

-- Trigger: Bump source token on page modification
create or replace function public.story_bump_page_source_token()
returns trigger language plpgsql set search_path = public, pg_temp as $$
begin
  if row(new.ocr_text, new.lesson, new.page_number) is distinct from row(old.ocr_text, old.lesson, old.page_number) then
    new.story_source_change_token := old.story_source_change_token + 1;
  end if;
  return new;
end; $$;

drop trigger if exists pages_story_bump_source_token on public.pages;
create trigger pages_story_bump_source_token before update of ocr_text, lesson, page_number on public.pages
for each row execute function public.story_bump_page_source_token();

-- Trigger: Smart Enqueue with Range Coalescing on Page Ready / Update
create or replace function public.story_enqueue_page_work()
returns trigger language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_start integer;
  v_end integer;
  v_existing_id uuid;
begin
  if new.status = 'ready' and (coalesce(new.ocr_text, '') <> '' or new.lesson is not null) then
    v_start := greatest(1, new.page_number - 1);
    v_end   := new.page_number + 1;

    -- Coalesce with any pending queued job for the same book whose range overlaps or touches
    select id into v_existing_id
      from public.story_jobs
     where book_id = new.book_id
       and job_type = 'process_range'
       and status = 'queued'
       and scope_start_page <= (v_end + 1)
       and scope_end_page >= (v_start - 1)
     limit 1 for update;

    if v_existing_id is not null then
      update public.story_jobs
         set scope_start_page = least(scope_start_page, v_start),
             scope_end_page   = greatest(scope_end_page, v_end),
             priority         = least(priority, 10::smallint),
             updated_at       = now()
       where id = v_existing_id;
    else
      insert into public.story_jobs
        (job_key, user_id, book_id, page_id, scope_start_page, scope_end_page, job_type,
         source_change_token, priority, status)
      values
        ('page:' || new.id::text || ':src:' || new.story_source_change_token::text || ':range:' || v_start::text || '-' || v_end::text,
         new.user_id, new.book_id, new.id, v_start, v_end, 'process_range',
         new.story_source_change_token, 10, 'queued')
      on conflict (job_key) do nothing;
    end if;
  end if;
  return new;
end; $$;

revoke all on function public.story_enqueue_page_work() from public, anon, authenticated;
drop trigger if exists pages_story_enqueue_work on public.pages;
create trigger pages_story_enqueue_work after insert or update of status, ocr_text, lesson, page_number on public.pages
for each row execute function public.story_enqueue_page_work();

-- Trigger: Invalidate segments and enqueue repair on page deletion or un-readiness
create or replace function public.story_handle_page_deletion()
returns trigger language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_start integer;
  v_end integer;
  v_seg_ids uuid[];
begin
  v_start := greatest(1, old.page_number - 1);
  v_end   := old.page_number + 1;

  -- Find all segments linked to this page
  select coalesce(array_agg(segment_id), '{}'::uuid[]) into v_seg_ids
    from public.story_segment_pages where page_id = old.id;

  if array_length(v_seg_ids, 1) > 0 then
    -- Mark segments inactive and invalidate their localizations immediately
    update public.story_segments set active = false, updated_at = now() where id = any(v_seg_ids);
    update public.story_segment_localizations
       set script_status = 'failed', audio_status = 'failed',
           script_source_hash = null, audio_source_hash = null,
           script_error = 'source_page_deleted', audio_error = 'source_page_deleted',
           updated_at = now()
     where segment_id = any(v_seg_ids);
  end if;

  -- Enqueue job to rebuild the neighboring range
  insert into public.story_jobs
    (job_key, user_id, book_id, scope_start_page, scope_end_page, job_type, priority, status)
  values
    ('repair_del:' || old.id::text || ':' || gen_random_uuid()::text,
     old.user_id, old.book_id, v_start, v_end, 'process_range', 10, 'queued');

  return old;
end; $$;

revoke all on function public.story_handle_page_deletion() from public, anon, authenticated;
drop trigger if exists pages_story_handle_deletion on public.pages;
create trigger pages_story_handle_deletion before delete on public.pages
for each row execute function public.story_handle_page_deletion();

-- Trigger: Enqueue storage cleanup when a book is deleted
create or replace function public.story_handle_book_deletion()
returns trigger language plpgsql security definer set search_path = public, pg_temp as $$
begin
  insert into public.story_jobs
    (job_key, user_id, book_id, job_type, priority, status, payload)
  values
    ('cleanup_book:' || old.id::text, old.user_id, old.id, 'cleanup_book_storage', 50, 'queued',
     jsonb_build_object('storage_prefix', old.user_id::text || '/' || old.id::text));
  return old;
end; $$;

revoke all on function public.story_handle_book_deletion() from public, anon, authenticated;
drop trigger if exists books_story_handle_deletion on public.books;
create trigger books_story_handle_deletion before delete on public.books
for each row execute function public.story_handle_book_deletion();

-- ----------------------------------------------------------------------------
-- WORKER RPCS (Service Role Only)
-- ----------------------------------------------------------------------------

-- Claim pending jobs with skip locked
create or replace function public.story_claim_jobs(p_limit integer default 3, p_lease_seconds integer default 300)
returns setof public.story_jobs language plpgsql security definer set search_path = public, pg_temp as $$
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs set status='failed', lease_token=null, lease_until=null,
    last_error_code=coalesce(last_error_code,'lease_expired_max_attempts'), updated_at=now()
   where status='processing' and lease_until <= clock_timestamp() and attempts >= max_attempts;
  return query
  with picked as (
    select id from public.story_jobs
     where attempts < max_attempts and available_at <= now()
       and (status in ('queued','retry') or (status='processing' and lease_until <= clock_timestamp()))
     order by priority, available_at, created_at
     for update skip locked
     limit greatest(1, least(coalesce(p_limit,3), 10))
  ), claimed as (
    update public.story_jobs j set status='processing', attempts=j.attempts+1,
      lease_token=gen_random_uuid(),
      lease_until=clock_timestamp()+make_interval(secs=>greatest(60,least(coalesce(p_lease_seconds,300),900))),
      heartbeat_at=clock_timestamp(), updated_at=now()
    from picked p where j.id=p.id returning j.*
  ) select * from claimed;
end; $$;

-- Renew active job lease
create or replace function public.story_renew_job_lease(p_job_id uuid, p_lease_token uuid, p_lease_seconds integer default 300)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs set lease_until=clock_timestamp()+make_interval(secs=>greatest(60,least(coalesce(p_lease_seconds,300),900))),
    heartbeat_at=clock_timestamp(), updated_at=now()
   where id=p_job_id and status='processing' and lease_token=p_lease_token and lease_until>clock_timestamp();
  get diagnostics n=row_count;
  return n=1;
end; $$;

-- Finish job with success/failure
create or replace function public.story_finish_job(
  p_job_id uuid, p_lease_token uuid, p_success boolean, p_error_code text default null, p_retry_after_seconds integer default 60
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs set
    status=case when p_success then 'succeeded' when attempts>=max_attempts then 'failed' else 'retry' end,
    available_at=case when not p_success and attempts<max_attempts then now()+make_interval(secs=>greatest(5,least(coalesce(p_retry_after_seconds,60),3600))) else available_at end,
    last_error_code=case when p_success then null else left(coalesce(p_error_code,'unknown'),120) end,
    lease_token=null, lease_until=null, updated_at=now()
   where id=p_job_id and status='processing' and lease_token=p_lease_token and lease_until>clock_timestamp();
  get diagnostics n=row_count;
  return n=1;
end; $$;

-- Guarded Atomic Segment Range Replacement
create or replace function public.story_replace_segment_range(
  p_job_id uuid, p_lease_token uuid, p_start_page integer, p_end_page integer, p_segments jsonb
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare
  j public.story_jobs;
  item jsonb;
  src jsonb;
  sid uuid;
  v_user uuid;
  v_old_ids uuid[];
  v_new_ids uuid[] := '{}'::uuid[];
  v_cur_hash text;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into j from public.story_jobs where id=p_job_id for update;
  if j.id is null or j.status<>'processing' or j.lease_token<>p_lease_token or j.lease_until<=clock_timestamp() then return false; end if;
  if p_start_page < 1 or p_end_page < p_start_page or jsonb_typeof(p_segments) <> 'array' then raise exception 'invalid range/payload'; end if;

  select user_id into v_user from public.books where id=j.book_id;
  if v_user is distinct from j.user_id then return false; end if;

  -- Lock book and affected pages before snapshot validation so source edits cannot race publish
  perform 1 from public.books where id=j.book_id and user_id=j.user_id for update;
  perform 1 from public.pages where book_id=j.book_id and page_number between p_start_page and p_end_page order by page_number for update;

  -- Identify segments currently mapped to pages in this range
  select coalesce(array_agg(distinct sp.segment_id),'{}'::uuid[]) into v_old_ids
    from public.story_segment_pages sp join public.pages p on p.id=sp.page_id
   where p.book_id=j.book_id and p.page_number between p_start_page and p_end_page;

  -- Guard: If any existing segment crosses boundary pages, refuse replacement (worker must expand range)
  if exists (
    select 1 from public.story_segment_pages sp join public.pages p on p.id=sp.page_id
     where sp.segment_id = any(v_old_ids) and (p.page_number < p_start_page or p.page_number > p_end_page)
  ) then return false; end if;

  -- Guard: Ensure NO segment in p_segments overwrites an existing segment OUTSIDE v_old_ids
  for item in select value from jsonb_array_elements(p_segments) loop
    if exists (
      select 1 from public.story_segments s
       where s.book_id = j.book_id and s.stable_key = item->>'stable_key'
         and not (s.id = any(v_old_ids))
    ) then
      raise exception 'segment key collision outside target range: %', item->>'stable_key';
    end if;

    -- Verify source page snapshots still match exactly
    if jsonb_typeof(item->'page_sources') <> 'array' or jsonb_array_length(item->'page_sources') = 0 then
      raise exception 'segment requires non-empty page_sources';
    end if;
    for src in select value from jsonb_array_elements(item->'page_sources') loop
      if not exists (
        select 1 from public.pages p
         where p.id = (src->>'page_id')::uuid and p.book_id = j.book_id and p.user_id = j.user_id
           and p.story_source_change_token = (src->>'source_change_token')::bigint
      ) then
        return false; -- Page source was modified concurrently
      end if;
    end loop;
  end loop;

  -- Detach mappings of old segments in range
  delete from public.story_segment_pages where segment_id = any(v_old_ids);

  -- Insert or update new segments
  for item in select value from jsonb_array_elements(p_segments) loop
    -- Check if existing segment hash changes, requiring immediate localization invalidation
    select source_hash into v_cur_hash from public.story_segments where book_id=j.book_id and stable_key=item->>'stable_key';

    insert into public.story_segments(user_id, book_id, segment_order, stable_key, source_hash, source_text, summary_text, facts, active, updated_at)
    values(j.user_id, j.book_id, (item->>'segment_order')::integer, item->>'stable_key', item->>'source_hash',
           item->>'source_text', coalesce(item->>'summary_text',''), coalesce(item->'facts','[]'::jsonb), true, now())
    on conflict(book_id, stable_key) do update set
      segment_order = excluded.segment_order,
      source_hash   = excluded.source_hash,
      source_text   = excluded.source_text,
      summary_text  = excluded.summary_text,
      facts         = excluded.facts,
      active        = true,
      updated_at    = now()
    returning id into sid;

    v_new_ids := array_append(v_new_ids, sid);

    -- Strict stale invalidation: if hash changed, invalidate localizations immediately!
    if v_cur_hash is not null and v_cur_hash <> (item->>'source_hash') then
      update public.story_segment_localizations
         set script_status = 'failed', audio_status = 'failed',
             script_source_hash = null, audio_source_hash = null,
             script_error = 'source_hash_changed', audio_error = 'source_hash_changed',
             updated_at = now()
       where segment_id = sid;
    end if;

    -- Attach fresh page mappings
    delete from public.story_segment_pages where segment_id = sid;
    for src in select value from jsonb_array_elements(item->'page_sources') loop
      insert into public.story_segment_pages(segment_id, page_id, source_order, source_change_token_snapshot)
      values(sid, (src->>'page_id')::uuid, (src->>'source_order')::integer, (src->>'source_change_token')::bigint);
    end loop;
  end loop;

  -- Soft delete / deactivate segments that were replaced and not present in new set
  update public.story_segments set active = false, updated_at = now()
   where id = any(v_old_ids) and not (id = any(v_new_ids));

  return true;
end; $$;

-- Atomic Publish of Script or Audio Stage
create or replace function public.story_publish_segment_stage(
  p_job_id uuid, p_lease_token uuid, p_segment_id uuid, p_language text, p_stage text, p_payload jsonb
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare
  j public.story_jobs;
  s public.story_segments;
  n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') or p_stage not in ('script','audio') then raise exception 'invalid stage/language'; end if;

  select * into j from public.story_jobs where id=p_job_id for update;
  if j.id is null or j.status<>'processing' or j.lease_token<>p_lease_token or j.lease_until<=clock_timestamp() then return false; end if;

  select * into s from public.story_segments where id=p_segment_id and active for update;
  if s.id is null or s.user_id<>j.user_id or s.book_id<>j.book_id then return false; end if;

  -- Validate pages have not changed since mapping
  perform 1 from public.pages p join public.story_segment_pages sp on sp.page_id=p.id
    where sp.segment_id=s.id order by p.id for update;
  if exists (
    select 1 from public.story_segment_pages sp join public.pages p on p.id=sp.page_id
     where sp.segment_id=s.id and p.story_source_change_token <> sp.source_change_token_snapshot
  ) then return false; end if;

  insert into public.story_segment_localizations(segment_id, language)
  values(p_segment_id, p_language) on conflict(segment_id, language) do nothing;

  if coalesce(p_payload->>'source_hash', '') <> s.source_hash then return false; end if;

  if p_stage = 'script' then
    update public.story_segment_localizations set
      script_status = 'ready',
      script_text   = p_payload->>'script_text',
      localized_summary = coalesce(p_payload->>'localized_summary', localized_summary),
      localized_facts   = coalesce(p_payload->'localized_facts', localized_facts),
      script_source_hash = p_payload->>'source_hash',
      script_error = null,
      updated_at = now()
    where segment_id = p_segment_id and language = p_language
      and nullif(p_payload->>'script_text', '') is not null;
  else
    -- Audio publish requires script to be ready and match current source hash
    if not exists (
      select 1 from public.story_segment_localizations l
       where l.segment_id = p_segment_id and l.language = p_language
         and l.script_status = 'ready' and l.script_source_hash = s.source_hash
    ) then
      raise exception 'script is not ready for current source hash';
    end if;

    if coalesce(p_payload->>'audio_path', '') !~ ('^' || j.user_id::text || '/' || j.book_id::text || '/' || p_segment_id::text || '/' || p_language || '/')
       or coalesce(p_payload->>'audio_mime_type', '') not in ('audio/wav', 'audio/mpeg', 'audio/ogg', 'audio/mp4')
       or coalesce((p_payload->>'audio_duration_ms')::integer, 0) <= 0 then
      raise exception 'invalid audio metadata or path';
    end if;

    update public.story_segment_localizations set
      audio_status      = 'ready',
      audio_path        = p_payload->>'audio_path',
      audio_mime_type   = p_payload->>'audio_mime_type',
      audio_duration_ms = (p_payload->>'audio_duration_ms')::integer,
      audio_source_hash = p_payload->>'source_hash',
      audio_error       = null,
      updated_at        = now()
    where segment_id = p_segment_id and language = p_language;
  end if;

  get diagnostics n = row_count;
  return n = 1;
end; $$;

-- ----------------------------------------------------------------------------
-- SERVER-SIDE PLAYBACK & RAG RPCS (Authenticated / Service Role Gated)
-- ----------------------------------------------------------------------------

-- Start playback session (service_role only, called by Server Function after auth check)
create or replace function public.story_start_playback(p_user_id uuid, p_book_id uuid, p_segment_id uuid, p_language text)
returns table(session_id uuid, resume_position_ms bigint, audio_duration_ms integer)
language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_duration integer;
  v_resume bigint;
  v_sid uuid;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;

  if not exists (
    select 1 from public.books b join public.story_segments s on s.book_id = b.id
     where b.id = p_book_id and b.user_id = p_user_id and s.id = p_segment_id and s.user_id = p_user_id and s.active
  ) then raise exception 'segment unavailable'; end if;

  -- Strict validation: audio must match CURRENT segment source hash
  select l.audio_duration_ms into v_duration
    from public.story_segment_localizations l
    join public.story_segments s on s.id = l.segment_id
   where l.segment_id = p_segment_id and l.language = p_language
     and l.audio_status = 'ready' and l.audio_path is not null
     and l.audio_source_hash = s.source_hash;

  if v_duration is null or v_duration <= 0 then raise exception 'audio not ready or stale'; end if;

  select case when segment_id = p_segment_id then coalesce(position_ms, 0) else 0 end
    into v_resume
    from public.story_playback_progress
   where user_id = p_user_id and book_id = p_book_id and language = p_language;

  v_resume := least(coalesce(v_resume, 0), v_duration);

  insert into public.story_playback_sessions
    (user_id, book_id, segment_id, language, audio_duration_ms, started_position_ms, max_position_ms)
  values
    (p_user_id, p_book_id, p_segment_id, p_language, v_duration, v_resume, v_resume)
  returning id into v_sid;

  return query select v_sid, v_resume, v_duration;
end; $$;

-- Checkpoint playback (service_role only, called by Server Function)
create or replace function public.story_checkpoint_playback(p_user_id uuid, p_session_id uuid, p_position_ms bigint, p_is_playing boolean)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare
  x public.story_playback_sessions;
  v_now timestamptz;
  v_elapsed bigint;
  v_delta bigint;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into x from public.story_playback_sessions where id = p_session_id and user_id = p_user_id for update;
  if x.id is null or x.completed_at is not null then return false; end if;

  v_now := clock_timestamp();
  v_elapsed := greatest(0, least(15000, (extract(epoch from (v_now - x.last_checkpoint_at)) * 1000)::bigint));

  if p_position_ms < 0 or p_position_ms > x.audio_duration_ms then return false; end if;
  -- Bound forward progress by wall-clock time + safety tolerance
  if p_position_ms > x.position_ms + v_elapsed + 1200 then return false; end if;

  v_delta := case when x.is_playing then v_elapsed else 0 end;

  update public.story_playback_sessions set
    position_ms         = p_position_ms,
    max_position_ms     = greatest(max_position_ms, p_position_ms),
    accumulated_play_ms = accumulated_play_ms + v_delta,
    last_checkpoint_at  = v_now,
    is_playing          = p_is_playing
  where id = x.id;

  insert into public.story_playback_progress(user_id, book_id, language, segment_id, position_ms, updated_at)
  values(x.user_id, x.book_id, x.language, x.segment_id, p_position_ms, now())
  on conflict(user_id, book_id, language) do update set
    segment_id = excluded.segment_id,
    position_ms = excluded.position_ms,
    updated_at = now();

  return true;
end; $$;

-- Complete playback (service_role only, verified monotonic playback duration)
create or replace function public.story_complete_playback(p_user_id uuid, p_session_id uuid)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare
  x public.story_playback_sessions;
  v_required bigint;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into x from public.story_playback_sessions where id = p_session_id and user_id = p_user_id for update;
  if x.id is null or x.completed_at is not null then return false; end if;

  v_required := greatest(0, (x.audio_duration_ms - x.started_position_ms) * 9 / 10 - 1500);

  if x.max_position_ms < x.audio_duration_ms - 1000
     or x.accumulated_play_ms < v_required
     or x.position_ms < x.audio_duration_ms - 1000 then
    return false;
  end if;

  update public.story_playback_sessions set completed_at = now() where id = x.id;

  insert into public.story_segment_listening(user_id, segment_id, language, completed_at, listened_ms)
  values(x.user_id, x.segment_id, x.language, now(), x.accumulated_play_ms)
  on conflict(user_id, segment_id, language) do update set
    completed_at = excluded.completed_at,
    listened_ms = greatest(public.story_segment_listening.listened_ms, excluded.listened_ms);

  return true;
end; $$;

-- Get audio path for signing (service_role only, checks hash matching)
create or replace function public.story_get_audio_for_signing(p_user_id uuid, p_segment_id uuid, p_language text)
returns table(audio_path text, audio_mime_type text, audio_duration_ms integer)
language plpgsql security definer set search_path = public, pg_temp as $$
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;

  return query
  select l.audio_path, l.audio_mime_type, l.audio_duration_ms
    from public.story_segment_localizations l
    join public.story_segments s on s.id = l.segment_id
    join public.books b on b.id = s.book_id
   where s.id = p_segment_id and s.active and s.user_id = p_user_id and b.user_id = p_user_id
     and l.language = p_language and l.audio_status = 'ready'
     and l.audio_source_hash = s.source_hash
     and l.audio_path like p_user_id::text || '/' || b.id::text || '/' || s.id::text || '/' || p_language || '/%'
   limit 1;
end; $$;

-- User-authorized Idempotent Preparation Request
create or replace function public.story_request_preparation(p_book_id uuid, p_page_id uuid, p_language text)
returns uuid language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_page public.pages;
  v_id uuid;
  v_key text;
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;

  select * into v_page from public.pages where id = p_page_id and book_id = p_book_id and user_id = auth.uid() and status = 'ready';
  if v_page.id is null then raise exception 'page unavailable'; end if;

  v_key := 'request:' || p_book_id::text || ':' || p_language || ':p' || v_page.page_number::text;

  insert into public.story_jobs
    (job_key, user_id, book_id, page_id, scope_start_page, scope_end_page, job_type, language,
     source_change_token, priority, status)
  values
    (v_key, auth.uid(), p_book_id, p_page_id, greatest(1, v_page.page_number - 1), v_page.page_number + 1,
     'process_range', p_language, v_page.story_source_change_token, 0, 'queued')
  on conflict (job_key) do update set
    priority = 0,
    available_at = now()
  where story_jobs.status in ('queued', 'retry')
  returning id into v_id;

  return v_id;
end; $$;

-- Manual Retry on Failed Job
create or replace function public.story_retry_failed_job(p_job_id uuid)
returns uuid language plpgsql security definer set search_path = public, pg_temp as $$
declare
  old_job public.story_jobs;
  new_id uuid;
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  select j.* into old_job from public.story_jobs j join public.books b on b.id = j.book_id
   where j.id = p_job_id and j.user_id = auth.uid() and b.user_id = auth.uid() and j.status = 'failed' for share of j;
  if old_job.id is null then raise exception 'failed job unavailable'; end if;

  insert into public.story_jobs
    (job_key, parent_job_id, user_id, book_id, page_id, segment_id, scope_start_page, scope_end_page,
     job_type, language, source_change_token, priority, status, max_attempts)
  values
    ('manual-retry:' || gen_random_uuid()::text, old_job.id, old_job.user_id, old_job.book_id, old_job.page_id, old_job.segment_id,
     old_job.scope_start_page, old_job.scope_end_page, old_job.job_type, old_job.language, old_job.source_change_token, 0, 'queued', old_job.max_attempts)
  returning id into new_id;

  return new_id;
end; $$;

-- Batch Page Reordering in a Single Atomic Transaction
create or replace function public.story_reorder_pages(p_book_id uuid, p_page_orders jsonb)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_item jsonb;
  v_page_id uuid;
  v_new_num integer;
  v_min_affected integer := 999999;
  v_max_affected integer := -1;
  v_old_num integer;
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  if not exists(select 1 from public.books where id = p_book_id and user_id = auth.uid()) then
    raise exception 'book unavailable';
  end if;

  -- Lock all pages of the book
  perform 1 from public.pages where book_id = p_book_id and user_id = auth.uid() order by page_number for update;

  -- Step 1: Temporarily negate page numbers to avoid unique constraint violations
  update public.pages set page_number = -page_number where book_id = p_book_id and user_id = auth.uid();

  -- Step 2: Apply new order and track min/max affected range
  for v_item in select value from jsonb_array_elements(p_page_orders) loop
    v_page_id := (v_item->>'page_id')::uuid;
    v_new_num := (v_item->>'new_page_number')::integer;

    select abs(page_number) into v_old_num from public.pages where id = v_page_id and book_id = p_book_id and user_id = auth.uid();
    if v_old_num is not null then
      v_min_affected := least(v_min_affected, v_old_num, v_new_num);
      v_max_affected := greatest(v_max_affected, v_old_num, v_new_num);

      update public.pages
         set page_number = v_new_num,
             story_source_change_token = story_source_change_token + 1
       where id = v_page_id and book_id = p_book_id and user_id = auth.uid();
    end if;
  end loop;

  -- Step 3: Restore any unmentioned pages that remained negative
  update public.pages set page_number = abs(page_number) where book_id = p_book_id and user_id = auth.uid() and page_number < 0;

  -- Step 4: Enqueue exactly ONE coalesced range job covering the entire affected span with boundaries
  if v_max_affected >= v_min_affected then
    insert into public.story_jobs
      (job_key, user_id, book_id, scope_start_page, scope_end_page, job_type, priority, status)
    values
      ('reorder:' || p_book_id::text || ':' || gen_random_uuid()::text,
       auth.uid(), p_book_id, greatest(1, v_min_affected - 1), v_max_affected + 1, 'process_range', 10, 'queued');
  end if;

  return true;
end; $$;

-- Multilingual RAG Search over Completed Segments (Decoupled from listening language)
create or replace function public.story_search_completed_segments(p_book_id uuid, p_query text, p_limit integer default 5)
returns table(segment_id uuid, segment_order integer, summary_text text, facts jsonb, rank real)
language plpgsql security definer set search_path = public, pg_temp as $$
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  if not exists(select 1 from public.books where id = p_book_id and user_id = auth.uid()) then raise exception 'book unavailable'; end if;

  -- Returns segments the user has COMPLETED listening to in ANY language, querying the canonical facts/summary
  return query
  select s.id, s.segment_order, s.summary_text, s.facts,
         ts_rank(s.search_document, websearch_to_tsquery('simple', p_query)) as r
    from public.story_segments s
    join (
      select distinct heard.segment_id
        from public.story_segment_listening heard
       where heard.user_id = auth.uid()
    ) comp on comp.segment_id = s.id
   where s.book_id = p_book_id and s.user_id = auth.uid() and s.active
     and s.search_document @@ websearch_to_tsquery('simple', p_query)
   order by r desc, s.segment_order
   limit greatest(1, least(coalesce(p_limit, 5), 10));
end; $$;

-- ----------------------------------------------------------------------------
-- STORAGE BUCKET & RLS POLICIES
-- ----------------------------------------------------------------------------

insert into storage.buckets(id, name, public) values('story-audio', 'story-audio', false)
on conflict(id) do nothing;

alter table public.story_segments enable row level security;
alter table public.story_segment_pages enable row level security;
alter table public.story_segment_localizations enable row level security;
alter table public.story_jobs enable row level security;
alter table public.story_playback_progress enable row level security;
alter table public.story_playback_sessions enable row level security;
alter table public.story_segment_listening enable row level security;

revoke all on public.story_segments, public.story_segment_pages, public.story_segment_localizations, public.story_jobs,
  public.story_playback_progress, public.story_playback_sessions, public.story_segment_listening from public, anon, authenticated;

grant select on public.story_segments, public.story_segment_pages, public.story_segment_localizations, public.story_jobs,
  public.story_playback_progress, public.story_playback_sessions, public.story_segment_listening to authenticated;

grant all on public.story_segments, public.story_segment_pages, public.story_segment_localizations, public.story_jobs,
  public.story_playback_progress, public.story_playback_sessions, public.story_segment_listening to service_role;

drop policy if exists story_segments_read_owner on public.story_segments;
create policy story_segments_read_owner on public.story_segments for select to authenticated using(user_id = auth.uid());

drop policy if exists story_segment_pages_read_owner on public.story_segment_pages;
create policy story_segment_pages_read_owner on public.story_segment_pages for select to authenticated
  using(exists(select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()));

drop policy if exists story_localizations_read_owner on public.story_segment_localizations;
create policy story_localizations_read_owner on public.story_segment_localizations for select to authenticated
  using(exists(select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()));

drop policy if exists story_jobs_read_owner on public.story_jobs;
create policy story_jobs_read_owner on public.story_jobs for select to authenticated using(user_id = auth.uid());

drop policy if exists story_progress_read_owner on public.story_playback_progress;
create policy story_progress_read_owner on public.story_playback_progress for select to authenticated
  using(user_id = auth.uid() and exists(select 1 from public.books b where b.id = book_id and b.user_id = auth.uid()));

drop policy if exists story_sessions_read_owner on public.story_playback_sessions;
create policy story_sessions_read_owner on public.story_playback_sessions for select to authenticated
  using(user_id = auth.uid() and exists(select 1 from public.books b where b.id = book_id and b.user_id = auth.uid()));

drop policy if exists story_listening_read_owner on public.story_segment_listening;
create policy story_listening_read_owner on public.story_segment_listening for select to authenticated
  using(user_id = auth.uid() and exists(select 1 from public.story_segments s where s.id = segment_id and s.user_id = auth.uid()));

-- Restrict RPC permissions
revoke all on function public.story_claim_jobs(integer, integer) from public, anon, authenticated;
revoke all on function public.story_renew_job_lease(uuid, uuid, integer) from public, anon, authenticated;
revoke all on function public.story_publish_segment_stage(uuid, uuid, uuid, text, text, jsonb) from public, anon, authenticated;
revoke all on function public.story_finish_job(uuid, uuid, boolean, text, integer) from public, anon, authenticated;
revoke all on function public.story_replace_segment_range(uuid, uuid, integer, integer, jsonb) from public, anon, authenticated;
revoke all on function public.story_start_playback(uuid, uuid, uuid, text) from public, anon, authenticated;
revoke all on function public.story_checkpoint_playback(uuid, uuid, bigint, boolean) from public, anon, authenticated;
revoke all on function public.story_complete_playback(uuid, uuid) from public, anon, authenticated;
revoke all on function public.story_get_audio_for_signing(uuid, uuid, text) from public, anon, authenticated;

revoke all on function public.story_request_preparation(uuid, uuid, text) from public, anon;
revoke all on function public.story_retry_failed_job(uuid) from public, anon;
revoke all on function public.story_reorder_pages(uuid, jsonb) from public, anon;
revoke all on function public.story_search_completed_segments(uuid, text, integer) from public, anon;

grant execute on function public.story_claim_jobs(integer, integer) to service_role;
grant execute on function public.story_renew_job_lease(uuid, uuid, integer) to service_role;
grant execute on function public.story_publish_segment_stage(uuid, uuid, uuid, text, text, jsonb) to service_role;
grant execute on function public.story_finish_job(uuid, uuid, boolean, text, integer) to service_role;
grant execute on function public.story_replace_segment_range(uuid, uuid, integer, integer, jsonb) to service_role;
grant execute on function public.story_start_playback(uuid, uuid, uuid, text) to service_role;
grant execute on function public.story_checkpoint_playback(uuid, uuid, bigint, boolean) to service_role;
grant execute on function public.story_complete_playback(uuid, uuid) to service_role;
grant execute on function public.story_get_audio_for_signing(uuid, uuid, text) to service_role;

grant execute on function public.story_request_preparation(uuid, uuid, text) to authenticated;
grant execute on function public.story_retry_failed_job(uuid) to authenticated;
grant execute on function public.story_reorder_pages(uuid, jsonb) to authenticated;
grant execute on function public.story_search_completed_segments(uuid, text, integer) to authenticated;

commit;
```

---

## 12. مراجعة المخاطر وحالات التسابق (Race Conditions Checklist)

1. **Stale Playback Race:** إذا كان المستخدم يستمع للمقطع وقام في تبويب آخر بتعديل OCR الصفحة، فبمجرد حفظ التعديل يزداد `story_source_change_token` وتبطل الدالة أي محاولة لطلب رابط توقيع جديد، بينما الجلسة القائمة تنتهي بسلام دون كتابة progress زائف على المقطع الجديد.
2. **Double Enqueue Race:** إذا أضيفت صفحتان متتاليتان في نفس الثانية، فإن آلية الـ Coalescing في Trigger تفحص الصفوف بحالة `FOR UPDATE` وتدمج النطاقين في نطاق واحد متكامل.
3. **Partial Range Reordering:** دالة `story_reorder_pages` تأخذ قفلاً كاملاً `FOR UPDATE` على جميع صفحات الكتاب، مما يمنع أي استدعاء متزامن من إدخال أرقام صفحات سالبة أو مكررة.
4. **Worker Crash with Storage Leaks:** إذا رفع العامل ملف MP3 وانهارت بيئة التنفيذ قبل استدعاء `story_publish_segment_stage`، يمر وقت الـ Lease وتصبح المهمة جاهزة للاسترداد. إذا أعيدت المهمة ونجحت، تحل محل المسار؛ وإذا فشلت تسجل مهمة `cleanup_audio` لحذف الملفات المعلقة.

---

## 13. خطة العمل والانتقال للتنفيذ المرحلي (Phased Rollout Plan)

بعد مراجعة هذه الوثيقة واعتمادها، سننتقل إلى التنفيذ عبر المراحل التالية دون إحداث أي ارتباك في التطبيق الحالي:

1. **المرحلة الأولى: اختبارات Schema وRPCs على بيئة Staging**
   - التحقق من عدم وجود تعارض مع أعمدة وجداول الإنتاج الحالية.
   - تشغيل الـ Migration على قاعدة اختبارية وتشغيل حالات اختبار القبول (Acceptance Tests).
2. **المرحلة الثانية: بناء وتثبيت الـ Worker والـ Voice Profiles**
   - تطوير الـ Server Function الخاصة بـ Worker وتقطيع المشاهد.
   - ضبط Gemini TTS مع الـ Voice Profiles الثلاثة (المصري، البولندي، الإنجليزي) والتحقق العملي من جودة وتطابق النطق.
3. **المرحلة الثالثة: مسارات الاستماع وتوقيع الروابط (Audio Playback & Signed URLs)**
   - ربط واجهة التطبيق بـ Server Functions للاستماع وحفظ التقدم.
   - التحقق من عمل الـ Checkpoints على أجهزة الموبايل (Safari iOS و Chrome Android).
4. **المرحلة الرابعة: محرك Mini-RAG والتسجيل الصوتي للسؤال (Push-to-Talk STT)**
   - بناء مكون تسجيل السؤال الصوتي وإرساله إلى Server Function.
   - ربط البحث في الحقائق المكتملة وتوليد إجابة شهرزاد بصوتها المطابق للغة الواجهة.
