# Bug Bounty URL Analysis 

## الهدف

الفكرة باختصار: نستخدم أوامر بسيطة عشان نطلع أهم الروابط اللي لقيناها ونرتبها حسب أهميتها، ونفحص الهيدرات (Headers) حقّ أهم الروابط عشان ندور على أي ثغرات أو مشاكل أمنية.

## الأدوات

الأوامر تعتمد على أدوات أساسية موجودة في أي نظام لينكس: Bash و Curl و Awk و Egrep و gospider.

---

## 0️⃣ الزحف باستخدام gospider

```bash
gospider -s https://subdomain -c 2 -d 2 -t 10 -a -r
```

- `-s` : تحديد الدومين أو subdomain
    
- `-c 2` : عدد الاتصالات المتوازية لكل request
    
- `-d 2` : عمق الزحف
    
- `-t 10` : مهلة لكل request بالثواني
    
- `-a` : يتتبع جميع الروابط في JS/HTML
    
- `-r` : يقتصر على الروابط الداخلية
    
- الناتج: ملف `gospider_all.txt`
    
بس بما انه عندنا الكثيير من الsubs لازم نسوي اداة تعالج لنا ذي المشكلة 
ببساطه رحت لل AI قلت له ياكبتن سوي لي الاداة وعطيته شروطي وبعد معاناه معاه بسبب انه دلخ طلعنا بنتيجه حلوه والادة برفعها في gihub يمديكم تحملونها 

> بعد الانتهاء، يكون جاهز للتنظيف والمعالجة.

---

## 1️⃣ تنظيف الروابط من التكرار

```bash
sort -u output/gospider_all.txt > output/gospider_all_unique.txt
```

- يرتب الروابط أبجديًا.
    
- يمسح أي رابط متكرر.
    
- الناتج: ملف `gospider_all_unique.txt` يحتوي كل رابط مرة واحدة.
    

---

## 2️⃣ استخراج روابط باراميتر وصفحات تسجيل الدخول/admin

```bash
# روابط تحتوي باراميتر
grep '?' output/gospider_all_unique.txt > output/urls_with_params.txt

# صفحات تسجيل الدخول أو admin
grep -Ei 'login|admin' output/gospider_all_unique.txt > output/login_admin_urls.txt

# دمج وإزالة التكرار
cat output/urls_with_params.txt output/login_admin_urls.txt | sort -u > output/param_and_login_sorted.txt
```

- `param_and_login_sorted.txt` يحتوي كل الروابط المهمة سواء فيها باراميتر أو صفحات login/admin.
    

---

## 3️⃣ تصفية الروابط الغير مهمة (root + favicons + ملفات ثابتة + الدومين الأساسي)

```bash
# استبدل root بالدومين الأساسي
grep -v "https://www.myfitnesspal.com" output/param_and_login_sorted.txt \
> output/param_and_login_sorted_filtered.txt
```

- يحذف الروابط التي:
    
    - هي نفس الدومين الأساسي (`root`)
        
    - أي أيقونات/صور صغيرة
        
    - أي رابط يبدأ بالدومين الأساسي
        
- الناتج: `param_and_login_final.txt` يحتوي فقط على الروابط المهمة للفحص.
    

---

## 7️⃣ إدارة الملفات بعد الفحص

### الملفات الأساسية للاحتفاظ بها:

- `param_and_login_final.txt`: جميع الروابط المهمة بعد التصفية.
    

### الملفات الوسيطة اللي ممكن حذفها:

- `gospider_all_unique.txt`
    
- `urls_with_params.txt`
    
- `login_admin_urls.txt`
    
- `param_and_login_sorted.txt`
    

> بهذه الطريقة، يكون كل شيء مرتب، الملفات المهمة محفوظة، والوسيطات تم تنظيفها لتقليل الفوضى.

---

## 8️⃣ نصائح عملية

- ابدأ بالروابط اللي فيها أكبر عدد من الباراميترات (أعلى 20–30 رابط).
    
- افحص صفحات admin/login يدويًا.
    
- سجل كل PoC في ملف `pocs.md` مع نوع الثغرة ومستوى الخطورة.
    
- راقب الهيدرات:
    
    - `Content-Security-Policy (CSP)`
        
    - `X-Frame-Options`
        
    - `Strict-Transport-Security (HSTS)`
        
    - `Set-Cookie` بدون `HttpOnly` أو `Secure` → مؤشر على مشاكل أمنية.
        
- التركيز أولًا على الروابط اللي تحتوي على أكثر باراميترات.
