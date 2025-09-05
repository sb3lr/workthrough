#استخدام dig و whois في اختبار الاختراق

## 1️⃣ أداة whois

**الوظيفة:** معرفة بيانات تسجيل الدومين (Registrar، تاريخ الإنشاء والانتهاء، NS، جهات الاتصال).

**لماذا مهمة لمختبر الاختراق:**

* معرفة صاحب الدومين ومعلومات الاتصال.
* التحقق من صلاحيات DNS.
* فهم البنية الأساسية للنطاق.
* البحث عن فرص Bug Bounty أو Social Engineering.

**أمر أساسي:**

```bash
whois example.com
```

**شرح عملي:**

* **Domain Name / Registry Domain ID** → لتأكيد أن الدومين موجود وحقيقي.
* **Registrar** → الشركة المسؤولة عن تسجيل الدومين.
* **Creation/Expiration Dates** → معرفة العمر وإمكانية انتهاء الدومين قريبًا.
* **Name Servers (NS)** → لمعرفة الخوادم الرسمية للنطاق.
* **Domain Status** → يوضح القيود مثل clientDeleteProhibited.

**مثال عملي:**

```bash
whois google.com
```

* يظهر كل NS، وتاريخ التسجيل، وحالة الحماية، ومعلومات الاتصال الرسمية.

---

## 2️⃣ أداة dig

**الوظيفة:** استعلام DNS لمعرفة كل أنواع السجلات (A, AAAA, MX, TXT, NS, CNAME, AXFR).

### 2.1 استعلام A Record

```bash
dig example.com A
```

* يعطي عنوان IPv4.

**مثال:**

```
example.com. 14 IN A 23.192.228.84
example.com. 14 IN A 23.192.228.80
```

* كمختبر: معرفة الـ IP لاستهداف السيرفرات واكتشاف بنية الشبكة.

### 2.2 استعلام AAAA Record

```bash
dig example.com AAAA
```

* يعطي عنوان IPv6.

### 2.3 استعلام MX Record

```bash
dig example.com MX
```

* يعطي سيرفرات البريد.

**مثال:**

```
example.com. 86400 IN MX 0 .
```

* كمختبر: تحليل إعدادات البريد وفرص Phishing.

### 2.4 استعلام TXT Record

```bash
dig example.com TXT
```

* يعرض SPF, DKIM, DMARC.

**مثال:**

```
example.com. 86400 IN TXT "v=spf1 -all"
example.com. 86400 IN TXT "_k2n1y4vw3qtb4skdx9e7dxt97qrmmq9"
```

* كشف ضعف حماية البريد.

### 2.5 استعلام NS Record

```bash
dig example.com NS
```

* يعرض خوادم DNS الرسمية.

**مثال:**

```
example.com. 71198 IN NS ns1.example.com.
example.com. 71198 IN NS b.iana-servers.net.
```

* معرفة NS قبل تجربة AXFR.

### 2.6 استعلام AXFR Record (Zone Transfer)

```bash
dig @ns1.example.com example.com AXFR
```

* يحاول نقل كامل سجلات الدومين من NS.

**مثال:**

* مؤمن: `; Transfer failed.`
* غير مؤمن:

```
example.com. 3600 IN A 192.168.1.10
mail.example.com. 3600 IN MX 10 mail.example.com.
www.example.com. 3600 IN CNAME example.com.
```

* كمختبر: استخراج كل Subdomains وMX وTXT وCNAME.

**ملاحظة:**
نعم، بشكل أدق يمكن تجربة كل NS:

```bash
dig @a.iana-servers.net example.com AXFR
```

* هذا يسمح بمحاولة نقل الـ Zone من كل NS متاح.
* بعض NS قد يكون مسموح بالAXFR وآخر لا.

---

## 3️⃣ الدمج بين whois و dig

* **whois** → معلومات قانونية وتقنية عن الدومين، مثل NS وRegistrar.
* **dig** → معلومات تشغيلية عن DNS وسجلاته.
* **معًا** → تبني خريطة كاملة للنطاق.

**الفائدة للمختبر:**

* معرفة كل بنية النطاق قبل أي محاولة استغلال.
* كشف الثغرات في DNS أو البريد.
* تجهيز تقرير Bug Bounty شامل.

---

### خلاصة

* استخدام whois وdig معًا يعطينا نظرة شاملة: من من يملك الدومين، NS الرسمي، كل السجلات المهمة، وإمكانية استغلال الثغرات في البريد أو DNS.
* AXFR المفتوح يعتبر ثغرة مباشرة.
* تحليل A, AAAA, MX, TXT مفيد لفهم بنية الشبكة وحماية البريد.
* تجربة AXFR على كل NS متاح يعطي أفضل نتائج لمختبر الاختراق.
