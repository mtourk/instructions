# احكي يا شهرزاد: المعمارية وMigration للمراجعة النهائية

**الإصدار 2 — مقترح فقط.** لم أعدّل التطبيق، ولم أشغّل SQL أو Migration، ولم تُحذف أو تُعدّل أي بيانات قائمة.

## 1. خلاصة القرارات المعدّلة

1. اعتماد **Database-backed Job Runner** داخل Postgres مع Atomic Claiming وLease وRetry، بلا Redis أو VM أو سيرفر مستقل. Cron للاسترداد، ومحاولة تشغيل فورية من Server Function بعد enqueue.
2. تقسيم القصة دلالياً على مستوى الكتاب، لا افتراض أن كل مشهد/مقطع ينتمي لصفحة واحدة. العلاقة بين `story_segments` و`pages` هي Many-to-Many في `story_segment_pages`.
3. سيناريو النص والصوت لهما حالتا معالجة مستقلتان. فشل الصوت لا يعيد إنشاء سيناريو ناجح.
4. البحث للإجابة عن أسئلة الطفلة يستخدم PostgreSQL Full-Text Search على حقائق/ملخصات المقاطع المكتملة فقط. لا Vector Database في الإصدار الأول.
5. لا توجد نسخ قديمة للسيناريو أو الصوت. `source_change_token` أرقام fencing تمنع الكتابة القديمة فقط، وليست Version History.
6. توليد الصوت يكون للغة المستخدمة فقط؛ العربية مصرية، والبولندية بلفظ صحيح، والإنجليزية طبيعية. التبديل إلى لغة أخرى يبقي Segment UUID ويبدأ الصوت المقابل من البداية.
7. لا ثقة في إشارة `audio ended` وحدها لتسجيل الاستماع. يلزم playback session مع تقدّم متحقق ووقت تشغيل معقول، مع التنبيه أن عميل ويب لا يمكنه إثبات أن إنساناً سمع الصوت فعلاً ضد عميل متعمد التلاعب.
8. قاعدة المشروع الحية تعذر فحصها في المراجعة السابقة. لهذا هذا Migration **مسودة للمراجعة والتدقيق** وليس ملفاً جاهزاً للتشغيل على الإنتاج قبل مقارنة الـschema والصلاحيات الحية.

## 2. الأدلة وحدود المراجعة

### ما أكدته من ملفات المشروع

- قاعدة المشروع مسجلة كـLovable Cloud/Postgres عبر Supabase، لكن قراءة الجداول الحية فشلت سابقاً برسالة أن قاعدة البيانات غير متاحة.
- ملفات migrations تصف `profiles` بهوية `id = auth.uid()` و`child_nickname`، وتربط الكتب والصفحات بـ`user_id`. لم يظهر فيها `child_id` مستقل أو تسجيل دخول للطفلة؛ لذلك التصميم يملك البيانات بحساب ولي الأمر، واسم الطفلة في الملف الشخصي ليس هوية Auth.
- جدول `pages` الحالي يتضمن `user_id`, `book_id`, `page_number`, `status`, `ocr_text`, `lesson`.
- كود TTS المباشر من Gemini يقرأ PCM ويضيف WAV header، بينما مسار Lovable Gateway يعيد البايتات كما هي ولا يفحص MIME/container، مع أن عميل التشغيل يعلن `audio/wav`. لا يجوز تخزين أي من المسارين قبل فحص الصيغة وتطبيعها.
- الكود الحالي يضبط gateway على `google/gemini-3.1-flash-tts-preview`، والمسار المباشر على `gemini-2.5-flash-preview-tts`. توثيق Gemini الرسمي الحالي يعرّف Gemini 3.8 Flash TTS وFlash-Lite TTS؛ ويذكر Flash للاستخدامات التي تحتاج إخراجاً تعبيرياً/كتباً صوتية وFlash-Lite للكلفة والسرعة. توثيق 3.8 يذكر أيضاً اختلافاً في صيغة metadata، لذا لا يكفي تبديل اسم الموديل. [Gemini 3.8 Flash TTS](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash-tts)
- لم يتم تنفيذ اختبار صوت حي عبر أي من الموفّرين أو اللغات. دعم العربية المصرية والبولندية والإنجليزية يظل شرط قبول تجريبياً، لا افتراضاً.

### Cloudflare/Supabase

لا أفترض مهلة HTTP ثابتة 30–60 ثانية. حدود Cloudflare تفرق بين وقت CPU والوقت الحائطي؛ خطة Standard تحدد عادة 30 ثانية CPU افتراضياً ويمكن أن تصل إلى 5 دقائق، كما أن حدود Cron/Queue Consumer تختلف. Supabase Edge Function توثق 150 ثانية wall-clock في المجاني و400 ثانية في المدفوع، مع 2 ثانية CPU لكل طلب و150 ثانية idle timeout. `waitUntil` لا يوفر دواماً أو استعادة بعد موت العامل. لذلك العمل المتين يبقى في Postgres، والـworker يعالج دفعات قصيرة. [Cloudflare limits](https://developers.cloudflare.com/workers/platform/limits/) · [Supabase Edge Function limits](https://supabase.com/docs/guides/functions/limits)

## 3. نموذج البيانات المعدل ومسؤولية كل جدول

| الكيان | المسؤولية ومفاتيحه |
|---|---|
| `books` الموجود | مصدر ترتيب صفحات الكتاب؛ المالك `user_id` هو حساب ولي الأمر. لا يحتاج هذا التصميم إضافة fence جديد للكتاب. |
| `pages` الموجود | مصدر OCR/lesson. يضاف `story_source_change_token` لتمييز تغييرات المحتوى دون حفظ نسخ. |
| `story_segments` جديد | مشهد/مقطع دلالي على مستوى الكتاب، UUID ثابت، ترتيب داخل الكتاب، حقائق موجزة ومحتوى بحث. لا يحتوي `page_id` مفرداً. |
| `story_segment_pages` جديد | Many-to-Many بين المقطع والصفحات التي يقتبس/يلخص منها، مع ترتيب المصدر داخل المقطع ولقطة fencing لكل صفحة. |
| `story_segment_localizations` جديد | صف لكل مقطع ولغة. `script_status` و`audio_status` منفصلان، مع نص السيناريو وملف الصوت ومعلومات MIME/duration. |
| `story_jobs` جديد | طابور دائم مع نوع المهمة، نطاق الصفحات، اللغة المطلوبة، الأولوية، المحاولات، Lease وfencing. السجل التشغيلي ليس تاريخ نسخ محتوى. |
| `story_playback_progress` جديد | موضع الاستماع المنفصل لكل مستخدم وكتاب ولغة؛ المفتاح الأساسي `(user_id, book_id, language)`. |
| `story_playback_sessions` جديد | جلسات استماع مؤقتة، تقدّم monotonic ووقت فعلي تقريبي، لمقاومة الإكمال العرضي/المبكر. |
| `story_segment_listening` جديد | سجل إكمال المقاطع فقط بعد تحقق قواعد جلسة الاستماع. هو مصدر Mini-RAG الآمن. |
| `story-audio` Storage bucket | خاص، مملوك لمسار حساب ولي الأمر؛ العامل وحده يرفع/يعدل/يحذف. رابط Signed مؤقت لا يُخزن. |

### مفاتيح وفهارس مهمة

- `story_segments.id` UUID مستقل، لا يعتمد على رقم الترتيب.
- `story_segment_pages` مفتاحه الأساسي `(segment_id, page_id)` مع `UNIQUE(segment_id, source_order)` وفهارس عكسية على `page_id` و`segment_id`.
- `story_segment_localizations` مفتاحه `(segment_id, language)`، واللغة ضمن `ar, pl, en`.
- `story_playback_progress` مفتاحه `(user_id, book_id, language)`، فيحفظ موضعاً مستقلاً لكل لغة.
- jobs لها فهرس جزئي على `(priority, available_at, created_at)` للـqueued/retry وآخر على `lease_until` للـexpired.
- بحث الحقائق والنصوص يستخدم `tsvector` مولداً بـ`simple`؛ لأن الكتاب بولندي والواجهة قد تكون عربية أو إنجليزية، ولأن `simple` لا يفترض stemming بلغة واحدة. لكل localization عمود tsvector مشتق من `script_text + localized_summary + localized_facts` باستخدام `simple`، لأن سؤال الطفلة بلغة الواجهة يجب أن يطابق نصاً مفهرساً باللغة نفسها. البحث بـ`websearch_to_tsquery('simple', ...)` ثم ترتيب rank. الاسترجاع يرشح أولاً سجلات `story_segment_listening` المكتملة فقط ثم FTS. لا توجد Vector DB أو embeddings في v1.

## 4. Jobs: إدراج، تشغيل، ضمانات، أولوية

### حالات Job

`queued → processing → succeeded`، أو `processing → retry → processing`، أو `processing → failed`. وتوجد `cancelled` عند إلغاء نطاق أصبح قديماً. انتهاء Lease يجعل المهمة مؤهلة للـclaim مجدداً ما لم تتجاوز المحاولات. صفوف jobs هي حالة تشغيل/أخطاء، وليست نسخاً من السيناريو.

### Priority

`0` = المقطع/الصفحة التالية التي طلب المستخدم سماعها وباللغة الحالية. `10` = أول صفحات كتاب جديد واللغة الحالية. `30` = باقي الصفحات للغة الحالية. `50` = تجهيز لغات أخرى أو أعمال منخفضة الأولوية. العامل يرتب تصاعدياً حسب priority ثم وقت الجاهزية والإنشاء؛ لا يضمن أولوية مطلقة إذا كانت هناك وظيفة محجوزة مسبقاً.

### التشغيل الفوري + Cron Recovery

- Trigger عند جاهزية صفحة، وعند تعديل OCR/lesson أو رقم الصفحة، يضيف Job موحداً قابلاً للإزالة/إعادة البناء لنطاق المشاهد المتأثر. لا يقوم Trigger بإجراء اتصال HTTP.
- Server Function بعد نجاح حفظ الصفحة/طلب الاستماع تستدعي endpoint داخلياً لتشغيل العامل فوراً؛ تمرر فقط طلب التشغيل، وليس سرّاً إلى المتصفح. إذا فشل النداء، تبقى المهمة في قاعدة البيانات.
- Supabase Cron يستدعي العامل دورياً كـRecovery لاسترداد queued أو expired leases. يمكن اختيار Cloudflare Cron بدلاً منه بعد فحص النشر الحقيقي، لكن لا يُفعّل مجدولان يتنافسان.
- العامل يطالب بحد أقصى صغير من الوظائف عبر `FOR UPDATE SKIP LOCKED` ويدون token وlease deadline؛ كل publish/renew/finish يشترط token صحيحاً والـlease لم تنته بعد.

### إعادة المحاولة اليدوية

`story_retry_failed_job(p_job_id)` يفحص أن المستخدم مالك الكتاب وأن المهمة `failed`، ثم ينشئ job جديداً بـUUID وjob_key فريد، ويحيل إلى المهمة الأصلية بـ`parent_job_id`. لا يحاول إعادة استخدام `job_key` القديم أو تغيير صف فاشل إلى queued. إعادة المحاولة لا تمسح سجلات أو بيانات قصة موجودة.

### Backfill آمن

بعد إنشاء triggers، ينفذ backfill واحد `INSERT ... SELECT` للصفحات الجاهزة ذات OCR/lesson، مع job_key حتمي يتضمن page ID وsource token والنطاق المتأثر. `ON CONFLICT DO NOTHING` مناسب للـbackfill فقط لمنع التكرار؛ وليس آلية إعادة المحاولة اليدوية. backfill يضيف وظائف ولا يحدّث/يحذف أي صف موجود.

### ترتيب الصفحات والتعديل

- تحديث `ocr_text` أو `lesson` يزيد `pages.story_source_change_token`.
- تعديل `page_number` يزيد `pages.story_source_change_token` ويطلب إعادة تجهيز نطاق الجيران حول الرقمين القديم والجديد؛ لأن حد المشهد الدلالي قد يتغير. مسودة الـrow-trigger الحالية مثال أولي ولا تغطي بعد النطاق القديم وتجميع إعادة الترتيب الجماعي، لذلك لا تعد migration نهائية.
- عند دمج/تعديل ترتيب صفحات متعددة في transaction، trigger statement-level هو الأفضل لتجميع نطاق التغيير مرة واحدة. مسودة SQL تتضمن trigger row-level بسيطاً لإثبات التغيير، ويجب استبداله/توسيعه إلى queue-range batching إذا كان التطبيق يعيد ترتيب مجموعة كبيرة دفعة واحدة.
- العامل يعيد بناء التقسيم في النطاق المتأثر مع سياق من الجار السابق والتالي. عند تغيّر `source_hash` يجب اعتبار النص/الصوت المحليين الحاليين غير صالحين فوراً، وعدم تقديمهما قبل مطابقة hash، ثم إعادة توليدهما وتنظيف الملف القديم بعد النشر. لا يحتفظ بالمحتوى القديم. يلزم تنفيذ/اختبار الترتيب الذري وإعادة الترتيب الجماعي قبل اعتماد الـSQL.

## 5. Atomic Publish وfencing

`story_publish_segment_stage` عملية ذرية واحدة، تستقبل job ID وlease token وsegment ID واللغة ومرحلة `script` أو `audio` وبيانات المرحلة. قبل أي كتابة تتحقق من:

1. المهمة `processing`، والـlease token مطابق، و`lease_until > clock_timestamp()`.
2. المقطع ما زال active، وملكية المستخدم/الكتاب متطابقة.
3. كل صفحة مرتبطة بالمقطع ما زالت تحمل `story_source_change_token` نفسه الذي سجل في `story_segment_pages`، وأن source token لكل صفحة مرتبطة لم يتغير منذ أخذ لقطة النطاق.
4. اللغة والـMIME والحالة صالحة؛ مرحلة audio تتطلب object path داخل مجلد المالك والكتاب والمقطع واللغة.

ثم تحدث فقط حقول المرحلة المطلوبة: نشر script يغير `script_status/script_text` ولا يمس حالة audio؛ نشر audio يغير `audio_status/audio_path/...` ولا يعيد توليد أو مسح script. عند أي فشل تحقق ترجع `published=false/stale=true` بلا كتابة. إذا كان العامل قد رفع ملفاً مؤقتاً قبل فشل publish، يحذف الملف المؤقت بنفسه عبر service role أو ينشئ cleanup job. لا يجوز للعميل امتلاك صلاحية رفع هذا الملف.

## 6. التغيير بلا Version History

- لا توجد جداول نسخ أو حقول `version_number` للمحتوى.
- `story_source_change_token` عداد منافسة فقط: يمنع نتيجة قديمة من النشر.
- `story_segment_pages.source_change_token_snapshot` لقطة fencing للمقارنة، ليست نسخة محتوى.
- عند إعادة تجهيز نطاق، الناتج الجديد يحل محل النص/الصوت الحاليين للجزء المتأثر فقط. لا يوجد استرجاع نسخة سابقة.
- يفضل حذف/أرشفة operational job errors وفق سياسة احتفاظ محدودة منفصلة؛ لا تحفظ نصوص صفحات كاملة في errors أو logs.

## 7. اللغات، TTS، ملفات الصوت

### اختيار الموديل

المقترح المبدئي بعد مراجعة التوثيق: استخدام `gemini-3.8-flash-tts` للأجزاء الحكائية ذات التعبير واللهجة، وتجربة `gemini-3.8-flash-lite-tts` للدفعات الكبيرة أو المقاطع القصيرة إذا اجتاز الجودة. هذا قرار مرشح لا ضمان جودة/إتاحة؛ التوثيق الرسمي يصف Flash للمخرجات التعبيرية والكتب الصوتية وLite للسرعة والكلفة. تحديث اسم الموديل وحده غير كافٍ: صيغة Gemini 3.8 تستخدم `speech_metadata` للأسلوب بدلاً من تضمين إرشادات الأسلوب كنص قد ينطق به النموذج. المسار المباشر والمسار Gateway يحتاجان contract واضحاً لكل استجابة.

### تحقق الصيغة إلزامي

1. كل موفر يرجع `mime_type` وبيانات audio container بحسب استجابته الفعلية.
2. لا نضع `audio/wav` افتراضياً. نتحقق من magic bytes/codec؛ PCM خام يتطلب WAV header صحيح مع sample rate/channels/bit depth المعروف، أما MP3/OGG/WAV جاهز فيخزن كما هو بعد فحص الصيغة.
3. نرفض أو نحول خرجاً لا يمكن تحديد صيغة تشغيله. نسجل mime والطول والمدة بعد decode/metadata inspection.
4. اختبار فعلي للغات `ar-EG`, `pl-PL`, `en` لكل مزود متاح، بما في ذلك الحروف البولندية `ą ć ę ł ń ó ś ź ż`, اللفظ، ثبات الصوت، بداية/نهاية التسجيل، وتشغيل الملف في متصفحات الهاتف والكمبيوتر.

### Storage

Bucket `story-audio` خاص. المسار المقترح `auth_owner_id/book_id/segment_uuid/language/file.ext`. لا تعتمد المسارات على هوية طفلة غير مسجلة. الواجهة لا تحصل على صلاحيات `INSERT/UPDATE/DELETE` في `storage.objects`. Server Function يتحقق من أن `auth.uid()` يملك الكتاب/المقطع، ثم ينشئ Signed URL للمقطع الجاري والتالي فقط. Signed URLs قصيرة العمر ولا تُخزن في قاعدة البيانات.

## 8. Speech-to-Text والتفاعل وسجل الاستماع

### استقبال سؤال الطفلة

الإصدار الأول يستعمل Gemini Live Agent، الذي يدعم تفريغاً نصياً لإدخال المستخدم ومحادثة متعددة اللغات؛ يوجد أيضاً Live Transcription مستقل إن احتجنا pipeline تفريغ منفصلاً. نبدأ بـLive Agent transcripts لتقليل المكونات، ونثبت لغة السؤال من لغة الواجهة. [Gemini Live API](https://ai.google.dev/gemini-api/docs/live-api) · [Live transcription](https://ai.google.dev/gemini-api/docs/live-api/live-transcribe).: `ar-EG`, `pl-PL`, `en-US` (أو locale مضبوط بعد اختبار). يطلب من جلسة Live إعادة/عرض transcript نصي مع الرد، وتُرسل تعليمات اللغة واللهجة صراحة. لا نعتمد `SpeechRecognition` في المتصفح كمسار وحيد، بسبب اختلاف الدعم بين المتصفحات. إذا تعذر فهم السؤال أو كان transcript منخفض الثقة، تقول شهرزاد بلغتها «ما سمعتش كويس، كرري السؤال» ولا تخمن. مستقبلًا يمكن إضافة Google Speech-to-Text منفصل، لكن لا نضيف خدمة/مفتاح جديد قبل اختبار Live للصوت واللغات الثلاث.

كل سؤال يُجاب عليه باستخدام FTS على المقاطع المكتملة فقط. أي مقطع لم يكمل تشغيله مستبعد، بما في ذلك الجزء الجاري. إذا كان السؤال عن شيء لا يوجد في المقاطع المكتملة، يوضح المساعد أنه لم يصل لهذا الجزء بعد، دون كشف الأحداث التالية.

### متى يعتبر المقطع مكتملاً؟

لا يكفي `HTMLAudioElement.ended` منفرداً. الواجهة تبدأ `story_playback_sessions`، ثم ترسل checkpoints دورية عبر Server Function محمية: segment/session ID، position_ms، delta من الوقت، playback state. السيرفر يقبل تقدماً متزايداً فقط، يحد معدل التقدم إلى الوقت الحقيقي تقريباً مع سماحية صغيرة، ويتحقق من مدة الصوت المعروفة. completion يتطلب: تشغيل بدأ من 0 أو موضع استئناف موثق، مجموع زمن تشغيل متراكم فوق حد duration (مثلاً 90% مع سماحية للـseek)، position وصل إلى آخر ثانية/نهاية الملف، وعدم وجود قفزة غير مبررة. بعد ذلك فقط ينفذ RPC server-side إدراج `story_segment_listening`. عند إغلاق التطبيق قد تحفظ نقاط التقدم لكن لا يوضع completed. هذا يمنع التسجيل العرضي، لكنه لا يثبت أن الطفلة كانت تستمع فعلاً؛ إثبات الحضور البشري غير ممكن من متصفح عادي دون قيود أكثر تدخلاً.

## 9. شرح وظائف SQL وTriggers وWorkers

| الاسم | المسؤولية | من يستدعيه/صلاحياته |
|---|---|---|
| `story_bump_page_source_token` Trigger | يزيد عداد fencing عند تعديل OCR/lesson فقط؛ لا يحفظ نسخة. | Trigger قبل تحديث الصفحة. |
| `story_bump_page_source_token` Trigger | يزيد عداد fencing عند OCR/lesson/page_number معاً؛ لا يخزن نسخة. | Trigger قبل التحديث. |
| `story_enqueue_page_work` Trigger | يدرج job أولي/نطاق مشاهد بعد جاهزية الصفحة أو تعديلها. لا يتصل بالشبكة. | Trigger، بحدود أقل صلاحية ممكنة. |
| `story_enqueue_book_backfill` SQL statement | يدرج jobs للصفحات القديمة الجاهزة بشكل idempotent. | مشغل قاعدة البيانات مرة بعد تثبيت triggers؛ في هذا المسود لا ينفذ تلقائياً. |
| `story_claim_jobs(limit, lease)` RPC | حجز ذري عبر SKIP LOCKED وتعيين lease/token/attempt. | service_role فقط، من Worker. |
| `story_renew_job_lease(job, token)` RPC | تمديد lease فقط قبل انتهائها ومع token صحيح. لا يسمح بإحياء lease منتهية. | service_role فقط. |
| `story_publish_segment_stage(...)` RPC | Atomic publish للسيناريو أو الصوت بعد lease/source fences/ownership. | service_role فقط. |
| `story_finish_job(job, token, outcome)` RPC | نجاح/فشل/retry فقط إذا lease لا تزال صالحة. المنتهية لا يمكن إنهاؤها. | service_role فقط. |
| `story_retry_failed_job(job)` RPC | إنشاء وظيفة جديدة لمهمة failed بعد تحقق الملكية، مع parent_job_id. | authenticated؛ يتحقق من auth.uid والكتاب. |
| `story_request_preparation(book, page?, language)` RPC | إنشاء high-priority job عند طلب المستخدم. | authenticated؛ يثبت أن المستخدم يملك الكتاب والصفحة. |
| `story_search_completed_segments(book, language, query)` RPC | FTS على الحقائق والنصوص من المقاطع المكتملة فقط. | authenticated؛ يتحقق من الملكية ويحدد حد نتائج. |
| `story_get_audio_for_signing` RPC | يعيد مسار الملف وmetadata فقط إذا ثبتت ملكية المستخدم والكتاب والمقطع واللغة؛ Server Function بعدها تصدر Signed URL. | service_role فقط عبر Server Function بعد التحقق من جلسة المستخدم. |
| `story_start_playback` / `story_checkpoint_playback` / `story_complete_playback` RPCs | إنشاء الجلسة، checkpoint monotonic محدود بزمن الجدار، حفظ progress بحسب اللغة، وتسجيل الإكمال بعد تجاوز checks فقط. | `service_role` فقط عبر Server Functions؛ تتحقق الوظيفة من auth المستخدم وتثبت أن owner ID مأخوذ من الجلسة وليس body العميل. |
| `Story Worker` Server/Edge Function | يعالج عدة وظائف بحدود صغيرة؛ يقرأ الصفحة/النص، يبني segmentation، يولد script، يطلب TTS، يرفع private audio، ينشر كل مرحلة ذرياً، ويجدد lease. | استدعاء فوري داخلي وCron Recovery؛ secret server-side. |
| `Cron Recovery` | يستدعي العامل بشكل دوري، ويضمن استرداد queued/expired leases بعد غلق المتصفح أو انقطاع request. | Supabase Cron أو Cloudflare Cron واحد فقط. |
| `Signed URL Server Function` | يثبت ملكية كتاب/مقطع للمستخدم ثم يصدر رابطاً قصير العمر للمقطع الحالي والقادم. | Authenticated. |

> أسماء RPC هنا أسماء التصميم المقترحة؛ يجب أن تتطابق نهائياً مع migration والكود قبل التنفيذ. لا أشغّل Cron ولا أضع secrets في SQL.

## 10. مراجعة RLS وGRANTS وهوية الحساب

- الهوية التي تدعمها migrations الحالية هي parent/auth owner (`auth.uid()`)، ولديه profile واحد و`child_nickname`. لا يوجد child auth identity في الملفات التي راجعتها. كل جداول القصة تحمل `user_id` للمالك الأب، وServer Function تتحقق من `books.user_id = auth.uid()` قبل الوصول.
- جداول النصوص/المقاطع/jobs لها RLS للقراءة الخاصة بالمالك. لا يحصل المتصفح على صلاحية كتابة في `story_segments`, localizations أو jobs.
- تقدم الاستماع وسجلات الإكمال وقراءة audio path تدار عبر Server Functions؛ الـRPCs المرتبطة بها تقبل `service_role` فقط، والوظيفة تتحقق من auth session وتستخدم user ID من الجلسة لا من مدخلات العميل. لا يحصل المتصفح على `INSERT/UPDATE` أو تنفيذ لهذه RPCs.
- Storage: لا سياسات read/insert/update/delete للمستخدمين العاديين. العامل يرفع ويحذف بصلاحية server role، والواجهة تستخدم Signed URL قصيراً بعد التحقق من المالك.
- يجب فحص أن endpoint لا يقبل `owner_id` مرسلاً من العميل دون مقارنة مع `auth.uid()`. لا نعامل `child_nickname` كـUUID أو نستخدمه في Storage path.
- لا يضاف grant عام لدوال `SECURITY DEFINER`. كل function تحدد `search_path` وتعمل revoke من `public, anon, authenticated` ثم grant للدور المطلوب. أي RPC للمستخدم يجب أن يتحقق من الملكية داخلياً.

## 11. اختبارات القبول والفشل

### Schema/Backfill/Ownership
1. تطبيق المسودة على نسخة staging من schema، لا Production، دون duplicate column/policy/function أو أي drop لجدول قائم.
2. فشل قراءة Live schema يعني تعليق التنفيذ حتى تصدير schema الإنتاج ومقارنته.
3. Backfill مرتين لا ينشئ duplicate jobs؛ الصفحات غير الجاهزة أو الفارغة لا تدخل queue.
4. حساب parent A لا يقرأ/يطلب signed URL لمقطع parent B حتى لو عرف UUID. لا تعتمد الاختبارات على هوية child منفصلة غير موجودة.
5. مستخدم authenticated لا يستطيع رفع/تعديل/حذف ملف story-audio؛ service worker يستطيع ذلك فقط.

### Worker والـLease
6. عاملان متزامنان لا يحصلان على job نفسه بسبب SKIP LOCKED.
7. renew بعد انتهاء lease يرجع false ولا يمددها. finish بعد انتهاء lease يرجع false ولا يغير حالة المهمة.
8. lease منتهية يعاد claim لها؛ العامل القديم لا يستطيع publish بعد أن يملك عامل آخر token جديداً.
9. source token تغير قبل publish: لا script ولا audio قديم يستبدل الجديد؛ الملف المؤقت ينظف.
10. فشل TTS بعد script ready: تبقى `script_status=ready`, تصبح `audio_status=retry/failed`; retry يعيد TTS فقط.
11. manual retry على failed ينشئ job جديداً، والطلب المكرر لا يعيد استخدام job_key الفاشل.
12. إغلاق المتصفح بعد enqueue لا يوقف job. فشل immediate invocation يعالج بواسطة Cron.
13. Cron لا ينشئ مجدولين أو jobs متنافسة؛ idempotency/fences تمنع التكرار الضار.

### Segmentation/RAG/الترتيب
14. مشهد يمتد على صفحتين يرتبط بالصفحتين عبر M2M ويحافظ على ترتيب المصدر.
15. تعديل OCR يعيد النطاق المتأثر وجيرانه دون إعادة الكتاب كله ودون الاحتفاظ بنسخة قديمة.
16. إعادة ترتيب صفحات تغيّر source tokens للصفحات المتأثرة وتعيد تجهيز حدود المشاهد المجاورة.
17. FTS لا يرجع إلا المقاطع المكتملة، مع بحث كلمات بولندية/عربية/إنجليزية باستخدام `simple`; لا يوجد أي نتيجة من segment جارٍ أو مستقبلي.
18. سؤال لا تجيب عنه المقاطع المكتملة يرد بأن الجزء لم يُسمع بعد؛ لا يسرّب أحداثاً لاحقة.

### Playback/اللغة/STT
19. تشغيل طويل ثم إغلاق التطبيق يحفظ position مستقلاً لكل لغة؛ الاستكمال بنفس اللغة من millisecond position المخزن.
20. تبديل اللغة يحفظ segment UUID ويبدأ localized audio من 0ms؛ لا ينسخ timestamp بين اللغات.
21. `ended` وحده أو seek للقفز إلى النهاية لا يسجل completion؛ الإكمال الصحيح فقط بعد checkpoints/time/duration rules.
22. بث سؤال واضح بالعربية المصرية والبولندية والإنجليزية ينتج transcript/رداً باللغة المطلوبة بعد اختبار عملي؛ السؤال غير الواضح يطلب التكرار ولا يخمّن.
23. اختبار على Safari iOS وChrome Android وChrome/Firefox Desktop، والمايك لا يفتح إلا بموافقة المستخدم.
24. عند سؤال واضح، Live transcription/transcript يلتقط اللغة المتوقعة (ar/pl/en)؛ عند تعذر التعرف لا يصدر جواباً مخترعاً.
25. FTS يجيب بلغة واجهة الطفل مع الاستناد إلى حقائق مكتملة حتى لو كانت لغة المصدر/الاستماع مختلفة، دون إظهار محتوى غير مكتمل.

### TTS/Storage/جودة المحتوى
26. لكل provider: assert MIME حقيقي، magic bytes/codec، sample rate، مدة audio، تشغيل قبل وبعد التخزين.
27. اختبار نطق الحروف البولندية المذكورة، العربية المصرية غير الفصحى، والإنجليزية. رفض provider إذا أرجع raw bytes بلا format metadata قابل للتحقق.
28. signed URL للمقطع الحالي والقادم يعمل قبل expiry ويفشل بعده، ولا يظهر في قاعدة البيانات.
29. كتاب 300 صفحة يجهز تدريجياً، والأجزاء الأولى للغة الواجهة أولوية؛ لا يرسل الكتاب كاملاً في طلب واحد.
30. تحقق بشري لعينة شخصيات وأحداث للتأكد من عدم اختلاق أسماء/أحداث، ولا يتم إعلان نجاح الدقة اعتماداً على test آلي فقط.

## 12. SQL Migration المقترح للمراجعة فقط

**مهم: لا تشغّل هذا النص.** هذه مسودة SQL مرجعية للمراجعة وليست Migration جاهزة أو مثبتة. لا تنفذ قبل تصدير schema قاعدة الإنتاج وفحص الإضافات والسياسات ومطابقة الأعمدة، ثم استكمال trigger إعادة الترتيب/Atomic replacement وإجراء اختبارات staging. النص الحالي مقصود به أن يكون additive، لكنه يضيف أعمدة إلى `pages` وينشئ جداول جديدة ويشير إلى bucket باسم محدد؛ يجب التأكد من عدم وجود تعارض بالاسم أو حالة bucket. لا يحذف صفوف الجداول الحالية عن قصد، لكن أي تعارض bucket/سياسات يجب أن يوقف التنفيذ. لا يحتوي Cron أو سر Worker. صُمم من migrations المتاحة، وليس من قراءة ناجحة للـLive schema، ولم يجر تشغيله أو تحليله على PostgreSQL.

> ملاحظة تدقيقية: SQL أدناه يغطي الشكل المقترح للجداول والـRPCs الأساسية فقط، لكنه غير مكتمل بما يكفي للتشغيل: يلزم إكمال التعامل مع نطاق الترتيب القديم/الجديد، نشر استبدال المشاهد دون تعارض `segment_order`، فحص جميع تواقيع RPC/الأعمدة على schema الفعلية، وسياسات Storage القائمة. تكامل `pg_net`/Cron وHTTP secret غير متضمن. إعادة ترتيب عدة صفحات يجب أن تتم transaction واحدة مع تجميع النطاقات، ولا يُعتمد trigger row-level الحالي لهذا الغرض.

```sql
begin;

-- Concurrency fences only. They do not store prior OCR or story versions.
alter table public.pages
  add column if not exists story_source_change_token bigint not null default 1;

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
    to_tsvector('simple', coalesce(source_text,'') || ' ' || coalesce(summary_text,'') || ' ' || coalesce(facts::text,''))
  ) stored,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (book_id, stable_key),
  constraint story_segments_book_order_uq unique (book_id, segment_order) deferrable initially deferred
);
create index if not exists story_segments_book_order_idx
  on public.story_segments(book_id, segment_order) where active;
create index if not exists story_segments_search_idx
  on public.story_segments using gin(search_document) where active;
create index if not exists story_segments_user_book_idx
  on public.story_segments(user_id, book_id);

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

create table if not exists public.story_segment_localizations (
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
  script_status text not null default 'queued'
    check (script_status in ('queued','processing','ready','retry','failed')),
  script_text text,
  localized_summary text not null default '',
  localized_facts jsonb not null default '[]'::jsonb,
  search_document tsvector generated always as (to_tsvector('simple',coalesce(script_text,'')||' '||coalesce(localized_summary,'')||' '||coalesce(localized_facts::text,''))) stored,
  script_source_hash text,
  script_error text,
  audio_status text not null default 'queued'
    check (audio_status in ('queued','processing','ready','retry','failed')),
  audio_path text,
  audio_mime_type text,
  audio_duration_ms integer check (audio_duration_ms is null or audio_duration_ms >= 0),
  audio_source_hash text,
  audio_error text,
  updated_at timestamptz not null default now(),
  primary key (segment_id, language)
);
create index if not exists story_localizations_search_idx on public.story_segment_localizations using gin(search_document);
create index if not exists story_localizations_audio_status_idx
  on public.story_segment_localizations(audio_status, updated_at);
create index if not exists story_localizations_script_status_idx
  on public.story_segment_localizations(script_status, updated_at);

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
  job_type text not null check (job_type in ('process_range','prepare_script','prepare_audio','cleanup_audio')),
  language text check (language is null or language in ('ar','pl','en')),
  source_change_token bigint,
  priority smallint not null default 30 check (priority between 0 and 100),
  status text not null default 'queued'
    check (status in ('queued','processing','retry','succeeded','failed','cancelled')),
  attempts integer not null default 0 check (attempts >= 0),
  max_attempts integer not null default 6 check (max_attempts between 1 and 12),
  available_at timestamptz not null default now(),
  lease_token uuid,
  lease_until timestamptz,
  heartbeat_at timestamptz,
  last_error_code text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check ((status = 'processing') = (lease_token is not null and lease_until is not null))
);
create index if not exists story_jobs_claim_idx on public.story_jobs(priority, available_at, created_at)
  where status in ('queued','retry');
create index if not exists story_jobs_expired_idx on public.story_jobs(lease_until)
  where status = 'processing';
create index if not exists story_jobs_owner_idx on public.story_jobs(user_id, created_at desc);

create table if not exists public.story_playback_progress (
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
  segment_id uuid references public.story_segments(id) on delete set null,
  position_ms bigint not null default 0 check (position_ms >= 0),
  updated_at timestamptz not null default now(),
  primary key (user_id, book_id, language)
);

create table if not exists public.story_playback_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  book_id uuid not null references public.books(id) on delete cascade,
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
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
create index if not exists story_playback_sessions_owner_idx
  on public.story_playback_sessions(user_id, book_id, created_at desc);

create table if not exists public.story_segment_listening (
  user_id uuid not null,
  segment_id uuid not null references public.story_segments(id) on delete cascade,
  language text not null check (language in ('ar','pl','en')),
  completed_at timestamptz not null default now(),
  listened_ms bigint not null check (listened_ms >= 0),
  -- Reserved shape for future sentence-level timing; v1 records segment completion only.
  timing_metadata jsonb not null default '{}'::jsonb,
  primary key (user_id, segment_id, language)
);
create index if not exists story_listening_completed_idx
  on public.story_segment_listening(user_id, language, completed_at desc);

-- Validate M:N ownership so a segment cannot reference another owner's page or book.
create or replace function public.story_validate_segment_page_owner()
returns trigger language plpgsql security definer set search_path = public, pg_temp as $$
declare s public.story_segments; p public.pages;
begin
  select * into s from public.story_segments where id = new.segment_id;
  select * into p from public.pages where id = new.page_id;
  if s.id is null or p.id is null or s.user_id <> p.user_id or s.book_id <> p.book_id then
    raise exception 'segment/page ownership or book mismatch';
  end if;
  return new;
end; $$;
revoke all on function public.story_validate_segment_page_owner() from public, anon, authenticated;
drop trigger if exists story_segment_pages_validate_owner on public.story_segment_pages;
create trigger story_segment_pages_validate_owner before insert or update on public.story_segment_pages
for each row execute function public.story_validate_segment_page_owner();

-- Source fencing: OCR/lesson/page-order changes increment the page token; no historical copies are stored.
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

-- Page-order changes also increment the page source token above; a neighbor-range job is enqueued below.

-- Durable enqueue only. Worker wake-up is performed by a Server Function after the DB commit.
create or replace function public.story_enqueue_page_work()
returns trigger language plpgsql security definer set search_path = public, pg_temp as $$
declare
  v_status text;
  v_ocr text;
begin
  v_status := new.status;
  v_ocr := coalesce(new.ocr_text, '');
  if v_status = 'ready' and (v_ocr <> '' or new.lesson is not null) then
    insert into public.story_jobs
      (job_key,user_id,book_id,page_id,scope_start_page,scope_end_page,job_type,
       source_change_token,priority,status)
    values
      ('page:'||new.id::text||':src:'||new.story_source_change_token::text||':range:'||greatest(1,new.page_number-1)::text||'-'||(new.page_number+1)::text,
       new.user_id,new.book_id,new.id,greatest(1,new.page_number-1),new.page_number+1,'process_range',
       new.story_source_change_token,10,'queued')
    on conflict (job_key) do nothing;
  end if;
  return new;
end; $$;
revoke all on function public.story_enqueue_page_work() from public, anon, authenticated;
drop trigger if exists pages_story_enqueue_work on public.pages;
create trigger pages_story_enqueue_work after insert or update of status, ocr_text, lesson, page_number on public.pages
for each row execute function public.story_enqueue_page_work();

-- Atomic claim. An expired lease is reclaimable, but never renewable/finishable by its old token.
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
     limit greatest(1,least(coalesce(p_limit,3),10))
  ), claimed as (
    update public.story_jobs j set status='processing', attempts=j.attempts+1,
      lease_token=gen_random_uuid(),
      lease_until=clock_timestamp()+make_interval(secs=>greatest(60,least(coalesce(p_lease_seconds,300),900))),
      heartbeat_at=clock_timestamp(), updated_at=now()
    from picked p where j.id=p.id returning j.*
  ) select * from claimed;
end; $$;

create or replace function public.story_renew_job_lease(p_job_id uuid,p_lease_token uuid,p_lease_seconds integer default 300)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs set lease_until=clock_timestamp()+make_interval(secs=>greatest(60,least(coalesce(p_lease_seconds,300),900))),
    heartbeat_at=clock_timestamp(),updated_at=now()
   where id=p_job_id and status='processing' and lease_token=p_lease_token and lease_until>clock_timestamp();
  get diagnostics n=row_count;
  return n=1;
end; $$;

-- Stage payload is JSON to keep script and audio publish as independent atomic operations.
create or replace function public.story_publish_segment_stage(
  p_job_id uuid,p_lease_token uuid,p_segment_id uuid,p_language text,p_stage text,p_payload jsonb
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare j public.story_jobs; s public.story_segments; n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') or p_stage not in ('script','audio') then raise exception 'invalid stage/language'; end if;
  select * into j from public.story_jobs where id=p_job_id for update;
  if j.id is null or j.status<>'processing' or j.lease_token<>p_lease_token or j.lease_until<=clock_timestamp() then return false; end if;
  select * into s from public.story_segments where id=p_segment_id and active for update;
  if s.id is null or s.user_id<>j.user_id or s.book_id<>j.book_id then return false; end if;
  perform 1 from public.pages p join public.story_segment_pages sp on sp.page_id=p.id
    where sp.segment_id=s.id order by p.id for update;
  if exists (
    select 1 from public.story_segment_pages sp
    join public.pages p on p.id=sp.page_id
    join public.books b on b.id=s.book_id
    where sp.segment_id=s.id
      and (p.story_source_change_token<>sp.source_change_token_snapshot
        )
  ) then return false; end if;
  if not exists(select 1 from public.story_segment_pages where segment_id=s.id) then return false; end if;

  insert into public.story_segment_localizations(segment_id,language)
  values(p_segment_id,p_language) on conflict(segment_id,language) do nothing;
  if coalesce(p_payload->>'source_hash','') <> s.source_hash then return false; end if;
  -- Consumers must require localization source hashes to match the current segment hash.
  -- A replacement worker must invalidate stale script/audio before exposing the regenerated segment.
  if p_stage='script' then
    update public.story_segment_localizations set
      script_status='ready', script_text=p_payload->>'script_text',
      script_source_hash=p_payload->>'source_hash', script_error=null, updated_at=now()
    where segment_id=p_segment_id and language=p_language
      and nullif(p_payload->>'script_text','') is not null;
  else
    if not exists(select 1 from public.story_segment_localizations l where l.segment_id=p_segment_id
      and l.language=p_language and l.script_status='ready' and l.script_source_hash=s.source_hash) then
      raise exception 'script is not ready for current source'; end if;
    if coalesce(p_payload->>'audio_path','') !~ ('^'||j.user_id::text||'/'||j.book_id::text||'/'||p_segment_id::text||'/'||p_language||'/')
       or coalesce(p_payload->>'audio_mime_type','') not in ('audio/wav','audio/mpeg','audio/ogg','audio/mp4')
       or coalesce((p_payload->>'audio_duration_ms')::integer,0)<=0 then raise exception 'invalid audio metadata/path'; end if;
    update public.story_segment_localizations set
      audio_status='ready',audio_path=p_payload->>'audio_path',audio_mime_type=p_payload->>'audio_mime_type',
      audio_duration_ms=(p_payload->>'audio_duration_ms')::integer,
      audio_source_hash=p_payload->>'source_hash',audio_error=null,updated_at=now()
    where segment_id=p_segment_id and language=p_language;
  end if;
  get diagnostics n=row_count;
  return n=1;
end; $$;

create or replace function public.story_finish_job(
  p_job_id uuid,p_lease_token uuid,p_success boolean,p_error_code text default null,p_retry_after_seconds integer default 60
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare n integer;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  update public.story_jobs set
    status=case when p_success then 'succeeded' when attempts>=max_attempts then 'failed' else 'retry' end,
    available_at=case when not p_success and attempts<max_attempts then now()+make_interval(secs=>greatest(5,least(coalesce(p_retry_after_seconds,60),3600))) else available_at end,
    last_error_code=case when p_success then null else left(coalesce(p_error_code,'unknown'),120) end,
    lease_token=null,lease_until=null,updated_at=now()
   where id=p_job_id and status='processing' and lease_token=p_lease_token and lease_until>clock_timestamp();
  get diagnostics n=row_count;
  return n=1;
end; $$;

-- User-authorized immediate priority request; Server Function should invoke the worker after commit.
create or replace function public.story_request_preparation(p_book_id uuid,p_page_id uuid,p_language text)
returns uuid language plpgsql security definer set search_path = public, pg_temp as $$
declare v_page public.pages; v_id uuid;
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;
  select * into v_page from public.pages where id=p_page_id and book_id=p_book_id and user_id=auth.uid() and status='ready';
  if v_page.id is null then raise exception 'page unavailable'; end if;
  insert into public.story_jobs(job_key,user_id,book_id,page_id,scope_start_page,scope_end_page,job_type,language,
    source_change_token,priority,status)
  values('request:'||gen_random_uuid()::text,auth.uid(),p_book_id,p_page_id,greatest(1,v_page.page_number-1),v_page.page_number+1,
    'process_range',p_language,v_page.story_source_change_token,0,'queued') returning id into v_id;
  return v_id;
end; $$;

-- Manual retry creates a fresh operational job; it never reopens the old failed row.
create or replace function public.story_retry_failed_job(p_job_id uuid)
returns uuid language plpgsql security definer set search_path = public, pg_temp as $$
declare old_job public.story_jobs; new_id uuid;
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  select j.* into old_job from public.story_jobs j join public.books b on b.id=j.book_id
   where j.id=p_job_id and j.user_id=auth.uid() and b.user_id=auth.uid() and j.status='failed' for share of j;
  if old_job.id is null then raise exception 'failed job unavailable'; end if;
  insert into public.story_jobs(job_key,parent_job_id,user_id,book_id,page_id,segment_id,scope_start_page,scope_end_page,
    job_type,language,source_change_token,priority,status,max_attempts)
  values('manual-retry:'||gen_random_uuid()::text,old_job.id,old_job.user_id,old_job.book_id,old_job.page_id,old_job.segment_id,
    old_job.scope_start_page,old_job.scope_end_page,old_job.job_type,old_job.language,old_job.source_change_token,0,'queued',old_job.max_attempts) returning id into new_id;
  return new_id;
end; $$;

-- FTS uses only listening rows marked completed. No current/future unfinished segment enters the result set.
create or replace function public.story_search_completed_segments(p_book_id uuid,p_language text,p_query text,p_limit integer default 5)
returns table(segment_id uuid,segment_order integer,summary_text text,facts jsonb,rank real)
language plpgsql security definer set search_path = public, pg_temp as $$
begin
  if auth.uid() is null then raise exception 'authentication required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;
  if not exists(select 1 from public.books where id=p_book_id and user_id=auth.uid()) then raise exception 'book unavailable'; end if;
  return query
  select s.id,s.segment_order,l.localized_summary,l.localized_facts,
    ts_rank(l.search_document,websearch_to_tsquery('simple',p_query)) as r
  from public.story_segments s
  join public.story_segment_listening heard on heard.segment_id=s.id and heard.user_id=auth.uid()
  join public.story_segment_localizations l on l.segment_id=s.id and l.language=p_language and l.script_status='ready'
  where s.book_id=p_book_id and s.user_id=auth.uid() and s.active
    and l.search_document @@ websearch_to_tsquery('simple',p_query)
  order by r desc,s.segment_order
  limit greatest(1,least(coalesce(p_limit,5),10));
end; $$;

-- Atomically replace derived segments in a bounded page range. Input shape:
-- [{stable_key,segment_order,source_hash,source_text,summary_text,facts,page_sources:[{page_id,source_order,source_change_token}]}]
-- The worker must pass the captured page source tokens. Stale work returns false.
create or replace function public.story_replace_segment_range(
  p_job_id uuid,p_lease_token uuid,p_start_page integer,p_end_page integer,p_segments jsonb
) returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare j public.story_jobs; item jsonb; src jsonb; sid uuid; v_user uuid; v_old_ids uuid[]; v_new_ids uuid[] := '{}'::uuid[];
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into j from public.story_jobs where id=p_job_id for update;
  if j.id is null or j.status<>'processing' or j.lease_token<>p_lease_token or j.lease_until<=clock_timestamp() then return false; end if;
  if p_start_page<1 or p_end_page<p_start_page or jsonb_typeof(p_segments)<>'array' then raise exception 'invalid range/payload'; end if;
  select user_id into v_user from public.books where id=j.book_id;
  if v_user is distinct from j.user_id then return false; end if;
  perform 1 from public.books where id=j.book_id and user_id=j.user_id for update;
  perform 1 from public.pages where book_id=j.book_id and page_number between p_start_page and p_end_page order by page_number for update;
  select coalesce(array_agg(distinct sp.segment_id),'{}'::uuid[]) into v_old_ids
    from public.story_segment_pages sp join public.pages p on p.id=sp.page_id
   where p.book_id=j.book_id and p.page_number between p_start_page and p_end_page;
  -- If an existing semantic segment crosses the range boundary, expand the job range and retry;
  -- do not detach its out-of-range source pages.
  if exists(select 1 from public.story_segment_pages sp join public.pages p on p.id=sp.page_id
    where sp.segment_id=any(v_old_ids) and (p.page_number<p_start_page or p.page_number>p_end_page)) then return false; end if;

  -- Every submitted source page must still exist, belong to this book, and retain its captured source token.
  for item in select value from jsonb_array_elements(p_segments) loop
    if jsonb_typeof(item->'page_sources')<>'array' or jsonb_array_length(item->'page_sources')=0 then raise exception 'segment needs page_sources'; end if;
    for src in select value from jsonb_array_elements(item->'page_sources') loop
      if not exists(select 1 from public.pages p where p.id=(src->>'page_id')::uuid and p.book_id=j.book_id
        and p.user_id=j.user_id and p.story_source_change_token=(src->>'source_change_token')::bigint) then return false; end if;
    end loop;
  end loop;

  -- Detach all mappings for segments intersecting the affected range. Payload must include each
  -- replacement scene's complete source-page set, including context pages on both boundaries.
  delete from public.story_segment_pages where segment_id=any(v_old_ids);

  for item in select value from jsonb_array_elements(p_segments) loop
    insert into public.story_segments(user_id,book_id,segment_order,stable_key,source_hash,source_text,summary_text,facts,active,updated_at)
    values(j.user_id,j.book_id,(item->>'segment_order')::integer,item->>'stable_key',item->>'source_hash',
      item->>'source_text',coalesce(item->>'summary_text',''),coalesce(item->'facts','[]'::jsonb),true,now())
    on conflict(book_id,stable_key) do update set
      segment_order=excluded.segment_order,source_hash=excluded.source_hash,source_text=excluded.source_text,
      summary_text=excluded.summary_text,facts=excluded.facts,active=true,updated_at=now()
    returning id into sid;
    v_new_ids:=array_append(v_new_ids,sid);
    delete from public.story_segment_pages where segment_id=sid;
    for src in select value from jsonb_array_elements(item->'page_sources') loop
      insert into public.story_segment_pages(segment_id,page_id,source_order,source_change_token_snapshot)
      values(sid,(src->>'page_id')::uuid,(src->>'source_order')::integer,
        (src->>'source_change_token')::bigint);
    end loop;
  end loop;
  delete from public.story_segments where id=any(v_old_ids) and not (id=any(v_new_ids));
  return true;
end; $$;

-- Server-only playback RPCs. The browser cannot directly write progress or completion rows.
create or replace function public.story_start_playback(p_user_id uuid,p_book_id uuid,p_segment_id uuid,p_language text)
returns table(session_id uuid,resume_position_ms bigint,audio_duration_ms integer)
language plpgsql security definer set search_path = public, pg_temp as $$
declare v_duration integer; v_resume bigint; v_sid uuid;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;
  if not exists(select 1 from public.books b join public.story_segments s on s.book_id=b.id
    where b.id=p_book_id and b.user_id=p_user_id and s.id=p_segment_id and s.user_id=p_user_id and s.active) then
    raise exception 'segment unavailable'; end if;
  select audio_duration_ms into v_duration from public.story_segment_localizations
    where segment_id=p_segment_id and language=p_language and audio_status='ready' and audio_path is not null;
  if v_duration is null or v_duration<=0 then raise exception 'audio not ready'; end if;
  select case when segment_id=p_segment_id then coalesce(position_ms,0) else 0 end
    into v_resume from public.story_playback_progress
    where user_id=p_user_id and book_id=p_book_id and language=p_language;
  v_resume:=least(coalesce(v_resume,0),v_duration);
  insert into public.story_playback_sessions(user_id,book_id,segment_id,language,audio_duration_ms,started_position_ms,max_position_ms)
  values(p_user_id,p_book_id,p_segment_id,p_language,v_duration,v_resume,v_resume) returning id into v_sid;
  return query select v_sid,v_resume,v_duration;
end; $$;

create or replace function public.story_checkpoint_playback(p_user_id uuid,p_session_id uuid,p_position_ms bigint,p_is_playing boolean)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare x public.story_playback_sessions; v_now timestamptz; v_elapsed bigint; v_delta bigint;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into x from public.story_playback_sessions where id=p_session_id and user_id=p_user_id for update;
  if x.id is null or x.completed_at is not null then return false; end if;
  v_now:=clock_timestamp();
  v_elapsed:=greatest(0,least(15000,(extract(epoch from (v_now-x.last_checkpoint_at))*1000)::bigint));
  if p_position_ms<0 or p_position_ms>x.audio_duration_ms then return false; end if;
  -- Bound forward seek by elapsed time, but permit backward seeks; exact resume position follows the latest checkpoint.
  if p_position_ms > x.position_ms + v_elapsed + 1200 then return false; end if;
  v_delta:=case when x.is_playing then v_elapsed else 0 end;
  update public.story_playback_sessions set position_ms=p_position_ms,
    max_position_ms=greatest(max_position_ms,p_position_ms),
    accumulated_play_ms=accumulated_play_ms+v_delta,last_checkpoint_at=v_now,is_playing=p_is_playing where id=x.id;
  insert into public.story_playback_progress(user_id,book_id,language,segment_id,position_ms,updated_at)
  values(x.user_id,x.book_id,x.language,x.segment_id,p_position_ms,now())
  on conflict(user_id,book_id,language) do update set segment_id=excluded.segment_id,
    position_ms=excluded.position_ms,updated_at=now();
  return true;
end; $$;

create or replace function public.story_complete_playback(p_user_id uuid,p_session_id uuid)
returns boolean language plpgsql security definer set search_path = public, pg_temp as $$
declare x public.story_playback_sessions; v_required bigint;
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  select * into x from public.story_playback_sessions where id=p_session_id and user_id=p_user_id for update;
  if x.id is null or x.completed_at is not null then return false; end if;
  v_required:=greatest(0,(x.audio_duration_ms-x.started_position_ms)*9/10-1500);
  if x.max_position_ms < x.audio_duration_ms-1000 or x.accumulated_play_ms < v_required or x.position_ms < x.audio_duration_ms-1000 then return false; end if;
  update public.story_playback_sessions set completed_at=now() where id=x.id;
  insert into public.story_segment_listening(user_id,segment_id,language,completed_at,listened_ms)
  values(x.user_id,x.segment_id,x.language,now(),x.accumulated_play_ms)
  on conflict(user_id,segment_id,language) do update set completed_at=excluded.completed_at,
    listened_ms=greatest(public.story_segment_listening.listened_ms,excluded.listened_ms);
  return true;
end; $$;

-- Server Function calls this ownership-checked RPC before creating a short-lived Storage signed URL.
create or replace function public.story_get_audio_for_signing(p_user_id uuid,p_segment_id uuid,p_language text)
returns table(audio_path text,audio_mime_type text,audio_duration_ms integer)
language plpgsql security definer set search_path = public, pg_temp as $$
begin
  if auth.role() <> 'service_role' then raise exception 'service_role required'; end if;
  if p_language not in ('ar','pl','en') then raise exception 'invalid language'; end if;
  return query
  select l.audio_path,l.audio_mime_type,l.audio_duration_ms
  from public.story_segment_localizations l
  join public.story_segments s on s.id=l.segment_id
  join public.books b on b.id=s.book_id
  where s.id=p_segment_id and s.active and s.user_id=p_user_id
    and b.user_id=p_user_id and l.language=p_language and l.audio_status='ready'
    and l.audio_path like p_user_id::text||'/'||b.id::text||'/'||s.id::text||'/'||p_language||'/%'
  limit 1;
end; $$;

-- Create private bucket only when absent. Do not silently change an existing bucket configuration.
insert into storage.buckets(id,name,public) values('story-audio','story-audio',false)
on conflict(id) do nothing;
-- Before deploy, verify any existing bucket is private and has no conflicting policies.
-- Deliberately no authenticated INSERT/UPDATE/DELETE policies. Service role owns writes/deletes.
-- No user-facing Storage policies: audio is served only by short-lived signed URLs from an ownership-checking Server Function.

-- RLS and grants: clients read own story text/status; all writes stay in trusted server RPCs/workers.
alter table public.story_segments enable row level security;
alter table public.story_segment_pages enable row level security;
alter table public.story_segment_localizations enable row level security;
alter table public.story_jobs enable row level security;
alter table public.story_playback_progress enable row level security;
alter table public.story_playback_sessions enable row level security;
alter table public.story_segment_listening enable row level security;

revoke all on public.story_segments,public.story_segment_pages,public.story_segment_localizations,public.story_jobs,
  public.story_playback_progress,public.story_playback_sessions,public.story_segment_listening from public,anon,authenticated;
grant select on public.story_segments,public.story_segment_pages,public.story_segment_localizations,public.story_jobs,
  public.story_playback_progress,public.story_playback_sessions,public.story_segment_listening to authenticated;
grant all on public.story_segments,public.story_segment_pages,public.story_segment_localizations,public.story_jobs,
  public.story_playback_progress,public.story_playback_sessions,public.story_segment_listening to service_role;

drop policy if exists story_segments_read_owner on public.story_segments;
create policy story_segments_read_owner on public.story_segments for select to authenticated using(user_id=auth.uid());
drop policy if exists story_segment_pages_read_owner on public.story_segment_pages;
create policy story_segment_pages_read_owner on public.story_segment_pages for select to authenticated
  using(exists(select 1 from public.story_segments s where s.id=segment_id and s.user_id=auth.uid()));
drop policy if exists story_localizations_read_owner on public.story_segment_localizations;
create policy story_localizations_read_owner on public.story_segment_localizations for select to authenticated
  using(exists(select 1 from public.story_segments s where s.id=segment_id and s.user_id=auth.uid()));
drop policy if exists story_jobs_read_owner on public.story_jobs;
create policy story_jobs_read_owner on public.story_jobs for select to authenticated using(user_id=auth.uid());
drop policy if exists story_progress_read_owner on public.story_playback_progress;
create policy story_progress_read_owner on public.story_playback_progress for select to authenticated
  using(user_id=auth.uid() and exists(select 1 from public.books b where b.id=book_id and b.user_id=auth.uid()));
drop policy if exists story_sessions_read_owner on public.story_playback_sessions;
create policy story_sessions_read_owner on public.story_playback_sessions for select to authenticated
  using(user_id=auth.uid() and exists(select 1 from public.books b where b.id=book_id and b.user_id=auth.uid()));
drop policy if exists story_listening_read_owner on public.story_segment_listening;
create policy story_listening_read_owner on public.story_segment_listening for select to authenticated
  using(user_id=auth.uid() and exists(select 1 from public.story_segments s where s.id=segment_id and s.user_id=auth.uid()));

-- Restrict every SECURITY DEFINER RPC. No execute by anon/authenticated except explicit user-facing RPCs.
revoke all on function public.story_claim_jobs(integer,integer) from public,anon,authenticated;
revoke all on function public.story_renew_job_lease(uuid,uuid,integer) from public,anon,authenticated;
revoke all on function public.story_publish_segment_stage(uuid,uuid,uuid,text,text,jsonb) from public,anon,authenticated;
revoke all on function public.story_finish_job(uuid,uuid,boolean,text,integer) from public,anon,authenticated;
revoke all on function public.story_replace_segment_range(uuid,uuid,integer,integer,jsonb) from public,anon,authenticated;
revoke all on function public.story_start_playback(uuid,uuid,uuid,text) from public,anon,authenticated;
revoke all on function public.story_checkpoint_playback(uuid,uuid,bigint,boolean) from public,anon,authenticated;
revoke all on function public.story_complete_playback(uuid,uuid) from public,anon,authenticated;
revoke all on function public.story_get_audio_for_signing(uuid,uuid,text) from public,anon,authenticated;
revoke all on function public.story_request_preparation(uuid,uuid,text) from public,anon;
revoke all on function public.story_retry_failed_job(uuid) from public,anon;
revoke all on function public.story_search_completed_segments(uuid,text,text,integer) from public,anon;
grant execute on function public.story_claim_jobs(integer,integer) to service_role;
grant execute on function public.story_renew_job_lease(uuid,uuid,integer) to service_role;
grant execute on function public.story_publish_segment_stage(uuid,uuid,uuid,text,text,jsonb) to service_role;
grant execute on function public.story_finish_job(uuid,uuid,boolean,text,integer) to service_role;
grant execute on function public.story_replace_segment_range(uuid,uuid,integer,integer,jsonb) to service_role;
grant execute on function public.story_start_playback(uuid,uuid,uuid,text) to service_role;
grant execute on function public.story_checkpoint_playback(uuid,uuid,bigint,boolean) to service_role;
grant execute on function public.story_complete_playback(uuid,uuid) to service_role;
grant execute on function public.story_get_audio_for_signing(uuid,uuid,text) to service_role;
grant execute on function public.story_request_preparation(uuid,uuid,text) to authenticated;
grant execute on function public.story_retry_failed_job(uuid) to authenticated;
grant execute on function public.story_search_completed_segments(uuid,text,text,integer) to authenticated;

-- Safe idempotent backfill: queues only eligible existing pages. Run after the triggers are installed,
-- and only after schema review/approval. This inserts jobs; it does not rewrite existing books/pages.
insert into public.story_jobs(job_key,user_id,book_id,page_id,scope_start_page,scope_end_page,job_type,
  source_change_token,priority,status)
select 'page:'||p.id::text||':src:'||p.story_source_change_token::text||':range:'||greatest(1,p.page_number-1)::text||'-'||(p.page_number+1)::text,
  p.user_id,p.book_id,p.id,greatest(1,p.page_number-1),p.page_number+1,'process_range',
  p.story_source_change_token,30,'queued'
from public.pages p join public.books b on b.id=p.book_id and b.user_id=p.user_id
where p.status='ready' and (coalesce(p.ocr_text,'')<>'' or p.lesson is not null)
on conflict(job_key) do nothing;

commit;
```

## 13. مراجعة نقدية للمسودة قبل أي تنفيذ

هذه المسودة توضح هيكل البيانات والـclaims/publish والحماية، لكنها تحتاج مراجعة migrations فعلية وstaging قبل الإنتاج. نقطتان يجب حسمهما في implementation SQL بعد schema live:

1. **Incomplete SQL draft:** لا تعتمد `story_replace_segment_range` قبل إعادة تصميم/اختبار إعادة استخدام المقاطع مع قيد ترتيب الكتاب، وإبطال localization القديم عند تغير source hash. كذلك يلزم trigger/عملية ذرية لنطاقي ترتيب الصفحات القديم والجديد؛ تنفيذ trigger الحالي لا يكفي.
2. **RLS policies:** قبل تشغيل يجب التأكد أن أسماء السياسات غير موجودة حالياً، أو تحويلها إلى `drop policy if exists` بأسماء التصميم فقط ثم إعادة إنشائها. لا تستخدم إسقاط سياسات عامة/قائمة ولا تغيّر السياسات القديمة.
3. **Playback RPCs:** الدوال start/checkpoint/complete مضمنة في المسودة وتأخذ owner ID من Server Function بعد التحقق من جلسة المستخدم، وتسمح لـservice_role فقط؛ لا تمنح المتصفح تنفيذ هذه RPCs مباشرة؛ اختبر عتباتها في staging قبل تفعيل listening/RAG.
4. **Atomic segment replacement:** دالة استبدال النطاق مضمنة وتتحقق من lease/fences وتحدّث M:N ضمن transaction واحدة. عامل segmentation يجب أن يرسل source snapshots الدقيقة لكل صفحة، مع اختبار rollback على أي صفحة stale.
5. **Reorder batching:** إعادة ترتيب مجموعة صفحات تحتاج endpoint/transaction تجمع النطاق مرة واحدة. المسودة trigger row-level قد تنتج أكثر من job عند تحديث أرقام متعددة؛ idempotency/fence تمنع النتائج القديمة لكنها لا تمنع كثرة jobs. يفضل statement-level trigger مع transition tables أو RPC إعادة ترتيب صفحات ذرية.
6. **Worker wake-up/Cron:** غير موجودين في SQL عمداً. بعد اختيار البيئة المتصلة الحية، يضاف إعداد Cron ومفتاحه بأداة secrets المناسبة، مع التأكد من عدم طباعته في SQL أو الواجهة.
7. **FTS multilingual:** `simple` حل خفيف لا يقوم بتطبيع جذري/اشتقاق لغوي؛ يلزم اختبار كلمات بولندية وعربية. هذا لا يضيف Vector DB، وإذا كانت النتائج غير كافية يمكن تحليلها قبل الانتقال لأي خدمة أخرى.
8. **TTS:** لا يمكن اعتماد اختيار 3.8 ولا تخزين أي صوت قبل تحديث صيغة الطلب والتحقق الحي من الاستجابة لكل provider واللغة. المسودة لا تدعي أن الاختبار العملي قد تم.

لذلك أتعامل مع النص على أنه **Migration مرجعي للمراجعة المعمارية** وليس جاهزاً للتشغيل على الإنتاج حتى مقارنة schema الإنتاج، اختبار جميع RPCs على staging، والتحقق من مزود TTS. العامل/توليد المحتوى/الـCron كود تشغيلي منفصل ولا ينشئه SQL وحده.

## 14. خارطة التنفيذ بعد اعتماد التصميم

1. استعادة/تصدير schema الإنتاج و`pg_policies`, grants, extensions, buckets، ثم diff مع migrations المتاحة؛ إذا تعذر الوصول، لا ننفذ.
2. استكمال Migration في فرع مراجعة: سياسات additive آمنة، Atomic segment-range replacement، playback RPCs، queue indexes، وإجراءات backfill/rollback غير المدمرة.
3. اختبارات SQL على نسخة staging: RLS، parent ownership، lease انتهاء/تجديد/إنهاء، stale publish، ترتيب صفحات، duplicate jobs، manual retry، backfill مرتين.
4. اختبار Gemini TTS 3.8 Flash مقابل Lite، والـgateway مقابل direct provider، باللغات الثلاث. تحديد MIME والتطبيع وتخزين private sample ثم تشغيله.
5. بناء Worker بعامل محدود concurrency، retries/backoff، fences، cleanup، immediate wake-up، وCron recovery.
6. توليد segmentation دلالي متعدد الصفحات ومحتوى FTS، ثم النص والصوت المستقلان لكل locale.
7. مشغل متدرج، Signed URLs للمقطع الحالي والقادم، استئناف اللغة، أسئلة Live/STT، وMini-RAG من completed segments فقط.
8. Backfill الكتب الحالية بعد مراقبة حمل قاعدة البيانات، مع progress وpause/resume للدفعات.
9. قبول ميداني على كتاب قصير ثم متوسط ثم كتاب كبير، ثم تفعيل تدريجي قبل تشغيل شامل.

## المراجع الرسمية

- [Gemini 3.8 Flash TTS وFlash-Lite](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash-tts)
- [Cloudflare Workers limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Supabase Edge Function limits](https://supabase.com/docs/guides/functions/limits)
- [Supabase PGMQ كبديل طابور Postgres جاهز](https://supabase.com/docs/guides/queues/pgmq)
