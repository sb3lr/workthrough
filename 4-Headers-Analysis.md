# فحص هيدرز أمان HTTP — دليل عملي ومباشر

هذا الملف يشرح عملياً **كيف تفحص هيدرز الأمان**، لماذا مهمين، أوامر عملية للحصول عليهم، كيفية تقييم الخطورة، ونماذج جاهزة للتقرير (يمكن لصقها مباشرة في البلاغ).

---

## أوامر سريعة لاستخراج الهيدرز
- HSTS:
```bash
curl -sI example.com | grep -i Strict-Transport-Security
```

- CSP:
```bash
curl -sI example.com | grep -i Content-Security-Policy
```

- X-Frame-Options:
```bash
curl -sI example.com | grep -i X-Frame-Options
```

- X-Content-Type-Options:
```bash
curl -sI example.com | grep -i X-Content-Type-Options
```

- Set-Cookie (مع عرض السطر التالي):
```bash
curl -sI example.com | grep -i Set-Cookie -A1
```

- Server / X-Powered-By:
```bash
curl -sI example.com | grep -i -E "Server|X-Powered-By"
```

---

## ماذا تبحث بالضبط وكيف تقيم الخطر (خطوة بخطوة)

### 1) HSTS — Strict-Transport-Security
- ماذا تبحث: وجود هيدر `Strict-Transport-Security` وقيمة `max-age` ووجود `includeSubDomains` و`preload`.
- تقييم خطورة:
  - **عالي**: غياب HSTS على صفحات المصادقة أو الدفع → معرضة لـ SSL-stripping.
  - **متوسط**: موجود لكن `max-age` صغير أو لا يستخدم `includeSubDomains`.
- فحص سريع:
```bash
curl -sI https://target | grep -i Strict-Transport-Security || echo "Missing HSTS"
```

### 2) Content-Security-Policy (CSP)
- ماذا تبحث: وجود CSP وما إذا احتوى `unsafe-inline` أو `*` أو `data:`.
- تقييم خطورة:
  - **عالي**: لا يوجد CSP أو يحتوي على `unsafe-inline`/`*` → يقلل حماية ضد XSS.
  - **متوسط**: CSP موجود لكنه واسع جدًا أو يسمح بمصادر غير موثوقة.
- ملاحظة عملية: إن وجدت انعكاس باراميتر مع XSS فتعتبر حالة حرجة حتى لو يوجد CSP ضعيف.

### 3) X-Frame-Options
- ماذا تبحث: وجود `DENY` أو `SAMEORIGIN` أو قيمة مفقودة.
- تقييم خطورة:
  - **عالي** على صفحات إدارة/دفع: غيابه يسمح بـ clickjacking.
  - **منخفض** في صفحات عامة غير حساسة.
- فحص سريع: `curl -sI target | grep -i X-Frame-Options || echo "Missing X-Frame-Options"`

### 4) X-Content-Type-Options: nosniff
- ماذا تبحث: وجود `X-Content-Type-Options: nosniff`.
- تقييم خطورة:
  - **عالي** إذا الموقع يسمح برفع ملفات والمفتقد → زيادة خطر تنفيذ ملفات خبيثة.
  - **منخفض** في مواقع لا تتعامل مع رفع ملفات.
- فحص سريع: `curl -sI target | grep -i X-Content-Type-Options || echo "Missing nosniff"`

### 5) Set-Cookie Flags (HttpOnly, Secure, SameSite)
- ماذا تبحث: كوكيز الجلسة (`session`, `auth`, `wp_logged_in`, `wordpress_sec`, ...) تحتوي على `HttpOnly; Secure; SameSite`.
- تقييم خطورة:
  - **عالي جداً**: أي session cookie بدون `HttpOnly` أو بدون `Secure` (على HTTPS) → تعرض سرقة الجلسة وCSRF.
- فحص عملي: استخدم `curl -sI` ثم راجع السطور التي تحتوي `Set-Cookie`. مثال للفلترة:
```bash
curl -sI https://target | grep -i Set-Cookie -A1
```

### 6) Server / X-Powered-By
- ماذا تبحث: الكشف عن نوع السيرفر/الفريمورك ورقم الإصدار.
- تقييم خطورة:
  - **متوسط**: مجرد الكشف ليس ضعفاً لكنه يساعد في استهداف CVE.
  - **عالي**: إذا الإصدار معروف فيه CVE — بلّغ فوراً مع مرجع CVE.
- خطوة عملية: لاحقًا قارن الإصدار مع قاعدة بيانات CVE (nvd, mitre, osv).

---

## نموذج تقييم سريع (لتضمينه في التقرير)

- **العنوان:** فحص هيدرز أمان HTTP
- **الهدف:** example.com
- **التاريخ:** 2025-09-26
- **الأدوات:** curl
- **الملخص:** تم فحص هيدرز الأمان الأساسية؛ النتائج التالية توضح نقاط الضعف وخطورتها.

**النتائج المفصّلة (مثال):**
- HSTS: **مفقود** — **خطر: عالي** (صفحة المصادقة). التوصية: تفعيل HSTS مع `max-age=31536000; includeSubDomains; preload`.
- CSP: **موجود لكن ضعيف** (`unsafe-inline` موجود) — **خطر: عالي**. التوصية: إزالة `unsafe-inline` واستبدالها بسياسة مصادر محددة.
- X-Frame-Options: **مفقود** — **خطر: عالي** على صفحات الإدارة. التوصية: إضافة `X-Frame-Options: DENY` أو `SAMEORIGIN`.
- X-Content-Type-Options: **موجود** — **خطر: منخفض**.
- Set-Cookie: `sessionid` بدون `HttpOnly` وبدون `Secure` — **خطر: عالي جداً**. التوصية: وضع `HttpOnly; Secure; SameSite=Lax` على كوكي الجلسة.
- Server: يكشف `Apache/2.4.29` — **خطر: متوسط**، مرجع CVE: (أضف مرجع CVE إن وجد).

---

## نصوص جاهزة لإرسال البلاغ (Copy-Paste)

- **عنوان البلاغ:** Missing HSTS on authentication pages (High)
- **الوصف:** During header inspection we found `Strict-Transport-Security` header is missing from authentication endpoints. This allows for HTTPS downgrade attacks (SSL-stripping) and session hijacking. Recommend enabling `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`.
- **الرفع التقني:** curl -sI https://example.com/login | grep -i Strict-Transport-Security

---

## نصائح عملية وأدوات مساعدة
- استخدم Burp لالتقاط الردود وفحص الهيدرز بشكل مرئي. وتمكين Intercept لرؤية Set-Cookie وقيمها.
- عند وجود رفع ملفات افحص MIME types وResponse headers بدقة.
- إن وجدت إصدار سيرفر غير محدث، طابقه مع قواعد البيانات (NVD/MITRE) قبل البلاغ.

---

## خاتمة سريعة
- ركّز على **كوكيز الجلسة** وصفحات **المصادقة/الدفع** أولاً.  
- غياب هيدر بسيط ممكن يصبح خطير جداً حسب السياق — قيّم الخطر بحسب نوع الصفحة والبيانات المعالجة.  
- استخدم النماذج في قسم "نصوص جاهزة" لتسريع كتابة البلاغات.

