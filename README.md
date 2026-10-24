# ENSTP SPORT

واجهة رياضية ثلاثية اللغات (العربية/الفرنسية/الإنجليزية)، مع الوضع الليلي، صفحة حجز، تسجيل طالب، قائمة طلباته، وصفحة إدارة منفصلة. المشروع لا يتصل بأي حساب Supabase أو Resend حتى تُكمل الإعدادات أدناه.

## محتويات المشروع

- `index.html`, `styles.css`, `app.js`: الموقع العام.
- `admin.html`, `admin.js`: لوحة المدير لمراجعة الطلبات.
- `config.example.js`: نموذج إعدادات الواجهة. انسخه إلى `config.js`.
- `supabase/schema.sql`: الجداول والسياسات والفهرسة والتحقق من تداخل الحجوزات المؤكدة.
- `supabase/functions/submit-booking`: التحقق من جلسة الطالب وحفظ الطلب وإشعار المدير.
- `supabase/functions/review-booking`: السماح للمدير فقط بالموافقة/الرفض وإشعار الطالب.
- `supabase/functions/cancel-booking`: إلغاء طلب الطالب المعلّق فقط.

## 1) إعداد Supabase

1. أنشئ مشروعًا في https://supabase.com/ واحفظ Project URL و anon/publishable key.
2. افتح **SQL Editor** وشغّل كامل محتوى `supabase/schema.sql`.
3. في إعدادات Authentication، فعّل Email/Password. اضبط Site URL و Redirect URLs إلى رابط الموقع المنشور، وأضف `http://localhost:5500` أو رابط التطوير الذي تستخدمه.
4. انسخ `config.example.js` إلى `config.js` وضع عنوان مشروعك ومفتاح anon. لا تضع `service_role` key في هذا الملف أبدًا.
5. أنشئ حساب المدير من صفحة `admin.html` أو من Authentication → Users. بعد إنشاء الحساب، انسخ UUID الخاص به.
6. في SQL Editor نفّذ الأمر التالي مع استبدال UUID:

```sql
insert into public.user_roles (user_id, role)
values ('UUID-OF-MANAGER-ACCOUNT', 'admin')
on conflict (user_id) do update set role = 'admin';
```

لا تمنح دور admin للطلبة. صلاحية المدير محفوظة في جدول `user_roles`، وتتحقق منها وظيفة `review-booking` على الخادم.

## 2) إعداد بريد Resend

1. أنشئ حسابًا في https://resend.com/ وأضف/تحقق من نطاق مرسل تملكه (مثال `sport.example.com`). اتبع سجلات DNS التي يعرضها Resend.
2. أنشئ API key في Resend.
3. ثبّت Supabase CLI وسجّل الدخول واربط المشروع:

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

4. اضبط أسرار الوظائف (لا تضعها في ملفات المتصفح أو GitHub):

```bash
supabase secrets set RESEND_API_KEY=re_xxxxxxxxx
supabase secrets set SPORTS_MANAGER_EMAIL=manager@example.com
supabase secrets set EMAIL_FROM="ENSTP SPORT <sport@your-verified-domain.com>"
```

يجب أن يكون `EMAIL_FROM` عنوانًا مسموحًا به ضمن النطاق الذي تحققت منه. عنوان Resend التجريبي ليس مناسبًا عادةً لإرسال رسائل إنتاج إلى مستلمين عشوائيين.

## 3) نشر وظائف Supabase

من مجلد المشروع:

```bash
supabase functions deploy submit-booking
supabase functions deploy review-booking
supabase functions deploy cancel-booking
```

أضف الأسرار اللازمة التي توفرها Supabase تلقائيًا للوظائف (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`) عبر بيئة الوظائف. لا تكشف service role key لأي عميل. إذا كان إعداد مشروعك لا يحقن هذه القيم تلقائيًا، أضفها كـ Supabase secrets من لوحة التحكم.

## 4) تشغيل الموقع

للتجربة المحلية، شغّل خادم ملفات ثابتة من مجلد المشروع، مثل Live Server في VS Code. لا تفتح الصفحة عبر `file://` لأن تدفق تسجيل الدخول وإعادة التوجيه يحتاجان أصل HTTP.

انشر المجلد على GitHub Pages أو استضافة ملفات ثابتة أخرى. ارفع `config.js` فقط إذا كانت إعداداته العامة صحيحة. مفتاح anon مخصص للمتصفح، لكنه لا يغني عن سياسات RLS.

## 5) كيف تعمل الحجوزات

- يسجل الطالب الدخول ثم يرسل طلبًا؛ يتم التحقق من الجلسة في وظيفة `submit-booking` وحفظ الطلب بحالة `pending`.
- يتلقى المدير إشعارًا بالبريد إذا تم إعداد Resend بشكل صحيح.
- يسجل المدير الدخول إلى `admin.html` ويوافق أو يرفض. تتحقق الوظيفة من دور المدير على الخادم.
- عند القرار، تُرسل رسالة للطالب إذا كان البريد مضبوطًا.
- يمكن للطالب إلغاء طلبه ما دام `pending` عبر وظيفة `cancel-booking`.
- يمنع مشغّل قاعدة البيانات تداخل الحجوزات ذات الحالة `approved` للملعب والتاريخ نفسيهما. الطلبات المعلقة لا تحجز الوقت نهائيًا؛ يمكن للمدير رفض الطلبات المتعارضة أثناء المراجعة.

## 6) فيديو المدير واللغات

زر «إعداد رابط الفيديو» يسمح بمعاينة رابط YouTube أو رابط ملف MP4 في هذا المتصفح، ويحفظه محليًا فقط. لإظهاره لكل الزوار، ضع الرابط في `config.js` في `videoUrl` وأعد النشر. استخدم رابطًا عامًا أو غير مدرج مسموحًا بالتضمين.

الترجمة تغطي الواجهة الأساسية؛ راجع المحتوى العربي/الفرنسي/الإنجليزي قبل النشر الرسمي، وأضف أسماء المنشآت والأوقات الحقيقية بعد اعتمادها من الإدارة. أسماء الرياضات المعروضة أمثلة وليست تأكيدًا بأن هذه المرافق متوفرة بالفعل.

## قبل الإطلاق الرسمي

- اختبر حجزين متعارضين، موافقة المدير، الرفض، والإلغاء بحساب طالب منفصل.
- اختبر رسائل البريد في مجلد spam وتحقق من نطاق المرسل.
- حدّث أسماء المرافق وساعات العمل وفق الواقع.
- أضف سياسة خصوصية ووسيلة تواصل إدارية معتمدة.
- راجع إعدادات Authentication وقيود المعدل (rate limits) وإعدادات CORS بما يناسب نطاقك. لا تعتمد على الواجهة وحدها للحماية.
