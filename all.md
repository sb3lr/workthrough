# workthrough
# Bug Bounty Workthrough - الحلقة الأولى

## 🎯 الهدف

* اختيار نطاق قانوني من Bug Bounty (HackerOne أو Bugcrowd).
* عمل فحص أساسي شامل: السيرفر، الهيدرات، DNS، ملفات robots.txt، واكتشاف Subdomains.
* تعليم المشاهدين لماذا كل خطوة مهمة وكيف تربط النتائج لاحقًا مع الثغرات.

---

## 1️⃣ اختيار النطاق

* افتح موقع Bugcrowd / HackerOne.
* اختار **برنامج Public**.
* مثال: `example.com` (بدّلها بالنطاق الذي تختاره).

💡 نصيحة: استخدم دومين عام أو مختبر مثل DVWA إذا أردت توضيح بدون كشف بيانات حساسة.

---

## 2️⃣ Whois + DNS Lookup

### Whois أمر:

```bash
whois example.com
```

### DNS Lookup أوامر:

```bash
nslookup example.com
```

```bash
dig example.com any
```

### 🔹 الفائدة:

* معرفة ملكية النطاق: اسم المسجل، الدولة، البريد الإلكتروني، وتاريخ التسجيل.
* جمع معلومات OSINT: ربط الإيميلات والدومينات المرتبطة بالشركة، مما يساعد في محاولات الهندسة الاجتماعية أو اكتشاف Subdomains.
* **A Record** → معرفة IP السيرفر، مفيد لاحقًا لفحص المنافذ والخدمات.
* **MX Record** → تحديد خوادم البريد التي قد تحتوي على ثغرات أو بيانات حساسة.
* **TXT Record** → فحص SPF/DKIM و CNAME، يساعد في التحقق من البريد والتحكم في Subdomains.
* **CNAME / Subdomains** → كشف الخدمات المخفية مثل staging, dev, api، مما يفتح فرص لفحص إضافي.

---

## 3️⃣ اكتشاف Subdomains

### أوامر:

```bash
subfinder -d example.com -o subdomains.txt
```

```bash
dnsx -d example.com -w /usr/share/wordlists/dns/dns-Jhaddix.txt -o dns.txt
```

### Subdomain Takeover Check:

```bash
subjack -w subdomains.txt -t 100 -timeout 30 -o takeover.txt
```

### 🔹 الفائدة:

* كل Subdomain يمثل نقطة دخول جديدة محتملة للاختراق.
* Subdomains مثل `dev` أو `staging` غالبًا أقل أمان لأنها تستخدم بيئات اختبارية أو إصدار قديم.
* Takeover Check يكشف إذا كانت أي Subdomain غير مستخدمة أو موجهة إلى خدمة خارجية، مما يتيح الاستيلاء عليها واستغلالها.
* جمع Subdomains يساعد في بناء خريطة شاملة للنطاق مما يسهل متابعة الثغرات لاحقًا.

---

## 4️⃣ التحقق من المواقع الحية

### أمر:

```bash
httpx -l subdomains.txt -o alive.txt -status-code -title -server -tech-detect
```

### 🔹 الفائدة:

* تحديد أي Subdomains حية (Status Code 200) أو محمية بكودات 401/403، وهذا يسمح بالتركيز على الأهداف القابلة للوصول.
* معرفة العناوين الحية تسهّل فحص CMS أو الهيدرات لكل موقع.
* يسهل اكتشاف الخدمات القديمة أو الإصدارات غير المحدثة التي تمثل ثغرات محتملة.

---

## 5️⃣ فحص السيرفر والهيدرات لكل Subdomain حي

### أوامر:

```bash
curl -I https://example.com
```

```bash
httpx -u https://example.com -title -tech-detect -status-code -server
```

### Banner Grabbing أعمق:

```bash
curl -I -s https://example.com | grep Server
nmap -sV -p 80,443 example.com
```

### Security Headers:

| الهيدر                           | القيمة المتوقعة                   | الفائدة               | نقصه يعني؟                         |
| -------------------------------- | --------------------------------- | --------------------- | ---------------------------------- |
| Strict-Transport-Security (HSTS) | max-age=31536000                  | يجبر HTTPS            | هجوم downgrade ممكن على HTTP       |
| X-Frame-Options                  | DENY / SAMEORIGIN                 | يمنع Clickjacking     | ممكن عمل iframe exploit            |
| Content-Security-Policy (CSP)    | default-src 'self'                | يقلل XSS              | أي XSS ممكن يكون حرِج              |
| X-Content-Type-Options           | nosniff                           | يمنع MIME sniffing    | رفع ملفات خبيثة ممكن تتنفذ         |
| Referrer-Policy                  | no-referrer-when-downgrade        | يحمي الـ URLs الحساسة | تسريب روابط وتوكنات                |
| Permissions-Policy               | camera=(), microphone=()          | يحدد صلاحيات          | مؤشر على عدم الحماية               |
| Set-Cookie                       | HttpOnly; Secure; SameSite=Strict | حماية الكوكيز         | غيابها = Session Hijacking أو CSRF |

### 🔹 الفائدة العامة:

* معرفة نوع السيرفر (nginx, Apache, cloudflare) يساعد على تحديد الهجمات المحتملة لكل نوع.
* تحليل الهيدرات يكشف الثغرات في الإعدادات الأمنية الأساسية.
* جمع المعلومات حول Cookies وSecurity Headers يساعد في منع Session Hijacking وXSS وClickjacking.

---

## 6️⃣ فحص robots.txt

### أمر:

```bash
curl https://example.com/robots.txt
```

### مثال نتيجة:

```
User-agent: *
Disallow: /admin
Disallow: /backup/
Disallow: /test/
```

### 🔹 الفائدة:

* كشف مسارات مخفية أو endpoints قد تحتوي ملفات حساسة.
* `/admin` → صفحة تسجيل دخول محتملة.
* `/backup/` → ملفات النسخ الاحتياطية.
* `/test/` → بيئات اختبارية قد تحتوي على ثغرات أو بيانات مؤقتة.
* يساعد في توجيه فحص Directory Fuzzing بشكل أكثر دقة.

---

## 7️⃣ Directory & File Fuzzing

### أوامر:

```bash
ffuf -u https://example.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -mc 200,403,401 -t 50
```

```bash
ffuf -u https://example.com/js/FUZZ -w /usr/share/seclists/Discovery/Web-Content/js.txt -mc 200
```

### 🔹 الفائدة:

* كشف صفحات admin أو backup أو test المخفية.
* العثور على ملفات JS قد تحتوي مفاتيح API أو endpoints حساسة.
* يساعد في اكتشاف نقاط دخول محتملة للثغرات مثل XSS أو LFI أو Auth Bypass.
* تحسين فهم بنية الموقع وربطها مع Subdomains الحية.

---

## 8️⃣ CMS & Framework Detection

```bash
whatweb https://example.com
wpscan --url https://example.com
```

### 🔹 الفائدة:

* معرفة نوع CMS أو framework المستخدم.
* تحديد الإصدارات القديمة أو المعروفة بثغراتها لتسهيل البحث عن Exploits.
* يساعد على استهداف الثغرات بدقة أكبر دون إضاعة الوقت على تقنيات غير مستخدمة.

---

## 9️⃣ APIs & Passive Recon

* API Endpoints:

```bash
httpx -l subdomains.txt -path /api -o api_endpoints.txt
```

* Passive Recon:

  * Google Dorking:

```
site:example.com inurl:admin
site:example.com filetype:env
```

* Shodan / Censys → كشف السيرفرات، open ports، SSL info.

### 🔹 الفائدة:

* كشف أي APIs أو واجهات قد تكون أقل حماية.
* Dorking يكشف المعلومات العامة المخفية على الإنترنت.
* Shodan / Censys يساعد في معرفة تكوين السيرفرات المفتوحة والضعف الأمني المحتمل.
* جميع النتائج تساعد في رسم خريطة شاملة للنطاق قبل البدء بالهجمات الفعلية.

---

## 🔟 Logging & Reporting Prep

* كل نتيجة تحفظها منظمة → لاحقًا تربط كل Subdomain أو endpoint مع نوع الثغرة المحتملة.
* استخدم CSV أو JSON:

```bash
httpx -l subdomains.txt -o alive.json -json
```

### 🔹 الفائدة:

* تسهيل متابعة الثغرات ومواقعها.
* يسهل مشاركة النتائج مع الفريق أو في تقرير Bug Bounty.
* يسمح بإعادة استخدام البيانات في عمليات فحص مستقبلية.

---

## 🔧 أدوات مساعدة للتحليل السريع

* **nmap** للهيدرات:

```bash
nmap --script http-security-headers -p 443 example.com
```

* **Python shcheck**:

```bash
python3 -m pip install shcheck
shcheck https://example.com
```

### 🔹 الفائدة:

* أدوات إضافية للتحقق من الهيدرات والخدمات بشكل سريع.
* توفير نظرة شاملة على مستوى الأمان لكل موقع.
