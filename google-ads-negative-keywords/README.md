# ورك فلو Google Ads – الاستبعاد الأوتوماتيكي للكلمات السلبية

ورك فلو n8n بيدخل على حساب Google Ads **كل ٣ أيام**، يجيب الـ **Search Terms**،
يقارنها بقائمة كلمات ممنوعة إنت رافعها في **Google Sheet**، واللي يطابق يضيفه
**Negative Keyword** أوتوماتيك على مستوى الحملة.

> فيه نسختين من الورك فلو:
> - **`workflow.json`** — النسخة الأساسية (قواعد فقط → استبعاد أوتوماتيك).
> - **`workflow-ai-suggest.json`** — النسخة بطبقة الذكاء الاصطناعي (ChatGPT) وضع **Suggest-only**
>   (بتكتب اقتراحات في تاب Review بدل ما تستبعد فعلياً). راجع قسم "طبقة الـ AI" في آخر الملف.

---

## 1) فكرة الشغل (Flow)

```
كل 3 أيام (Schedule)
      ↓
الإعدادات (Config)  ← هنا بتحط الـ IDs والتوكن وإعدادات المطابقة
      ↓
قراءة قائمة الكلمات من Google Sheet  →  تجميع الكلمات
      ↓
جلب Search Terms من Google Ads API (GAQL)
      ↓
Code: المطابقة وبناء العمليات  (يقارن كل search term بالكلمات الممنوعة)
      ↓
IF: فيه كلمات للاستبعاد؟
   ├─ نعم →  إضافة Negative Keywords (campaignCriteria:mutate)
   │         + تفكيك التقرير → تسجيل النتائج في شيت Log
   └─ لا  →  مفيش كلمات جديدة (وقوف)
```

---

## 2) المتطلبات قبل ما تشغّل

### أ) وصول Google Ads API
لازم يكون عندك:
1. **Developer Token** معتمد (من MCC → Tools → API Center). لو لسه Test مش هيشتغل على حساب حقيقي، لازم Basic/Standard access.
2. **OAuth2 Client** (Client ID + Client Secret) من Google Cloud Console، مفعّل عليه **Google Ads API**.
3. **Customer ID** بتاع الحساب المستهدف (١٠ أرقام من غير شرطات).
4. لو بتدخل عن طريق حساب مدير (**MCC**): الـ **login-customer-id** = رقم الـ MCC.

> نصيحة: خلّي الـ scope وقت الـ OAuth =
> `https://www.googleapis.com/auth/adwords`

### ب) الكريدنشيالز في n8n
اعمل اتنين credentials:

| الكريدنشيال | النوع في n8n | بيتستخدم فين |
|---|---|---|
| **Google Ads OAuth2** | `Google OAuth2 API` (Generic OAuth2) — Scope: `adwords` | نودات الـ HTTP Request |
| **Google Sheets account** | `Google Sheets OAuth2 API` | قراءة الكلمات + تسجيل الـ Log |

### ج) Google Sheet
اعمل شيت فيه تابين:
- **تاب `Keywords`**: عمود واحد (مثلاً `keyword`) وتحته الكلمات الممنوعة، كلمة في كل صف.
- **تاب `Log`**: سيبه فاضي، الورك فلو هيكتب فيه التقرير تلقائي (search term / matched word / negative / campaign / clicks / cost / date).

---

## 3) الاستيراد والتظبيط

1. في n8n: **Import from File** واختار `workflow.json`.
2. افتح نود **الإعدادات (Config)** وعدّل القيم:

| الحقل | معناه | القيمة الافتراضية |
|---|---|---|
| `customerId` | رقم حساب Google Ads (بدون شرطات) | `1234567890` |
| `loginCustomerId` | رقم الـ MCC (لو بتدخل من حساب مدير)؛ لو مفيش MCC حطّه نفس `customerId` | `1234567890` |
| `developerToken` | الـ Developer Token بتاعك | `YOUR_DEVELOPER_TOKEN` |
| `matchMode` | `contains` (السيرش تيرم فيه الكلمة) أو `exact` | `contains` |
| `negateMode` | `searchTerm` (يستبعد السيرش تيرم نفسه) أو `keyword` (يستبعد الكلمة الممنوعة نفسها) | `searchTerm` |
| `negativeMatchType` | `EXACT` / `PHRASE` / `BROAD` | `EXACT` |
| `minClicks` | حد أدنى للكليكات عشان يتعامل مع السيرش تيرم (0 = الكل) | `0` |
| `dateRange` | نطاق سحب البيانات | `LAST_14_DAYS` |

3. في النودات: **قراءة قائمة الكلمات** و **تسجيل النتائج** — حط `YOUR_GOOGLE_SHEET_ID`
   واختار الكريدنشيال بتاع Google Sheets.
4. في نودات الـ HTTP (**جلب Search Terms** و **إضافة Negative Keywords**) — اختار كريدنشيال
   **Google Ads OAuth2**.
5. شغّل **Execute Workflow** يدوي مرة للتجربة، وبعد ما تطمّن فعّل الـ **Active**.

---

## 4) ملاحظات مهمة

- **`partialFailure: true`** مفعّل في نود الإضافة، يعني لو كلمة سلبية موجودة قبل كده
  أو حصل خطأ في عملية واحدة، الباقي بيتنفّذ عادي من غير ما الـ batch كله يفشل.
- **إصدار الـ API**: الرابط بيستخدم `v18`. لو جوجل حدّثت الإصدار، غيّر `v18` في
  النودين للـ HTTP للإصدار الجديد.
- **negateMode = keyword أذكى للتنظيف الوقائي**: بيضيف الكلمة الممنوعة نفسها كـ
  negative (يفضّل `PHRASE`)، فبيمنع أي سيرش تيرم جديد فيه الكلمة دي مستقبلاً —
  مش بس اللي ظهر النهارده.
- **الجدولة**: النود متظبط كل ٣ أيام الساعة ٦ صباحاً. غيّرها من نود **Schedule**.
- **الأمان**: متحطّش الـ Developer Token في الشيت أو أي مكان عام؛ سيبه في نود Config
  أو الأفضل استخدم n8n Environment Variables.

---

## 5) لو مستوى الاستبعاد المطلوب مختلف

- **قائمة سلبية مشتركة (Shared Negative List)**: بدّل نود الإضافة لـ
  `sharedCriteria:mutate` وربط الحملات بالقائمة.
- **مستوى المجموعة (Ad Group)**: بدّل الـ URL لـ `adGroupCriteria:mutate`
  وفي الـ Code بدّل `campaign` بـ `adGroup` باستخدام `ad_group.id`
  (موجود أصلاً في استعلام الـ GAQL).

---

## 6) طبقة الذكاء الاصطناعي (ChatGPT) — نسخة `workflow-ai-suggest.json`

### الفكرة
بدل ما نعتمد على المطابقة الحرفية بس، ضفنا طبقة **ChatGPT** بتفهم **نيّة** كل search term
وتقرّر لو مناسب لنشاط الشركة ولا لأ، وتديك **سبب + نسبة ثقة (confidence)**.

الوضع الحالي **Suggest-only**: مفيش استبعاد فعلي على الحساب — كل الاقتراحات بتتكتب في
تاب **`Review`** عشان تراجعها بإيدك. بعد ما تطمن على جودة القرارات نفعّل الاستبعاد الأوتوماتيكي.

### الفلو
```
Schedule (كل 3 أيام)
  → Config (فيه businessContext = وصف نشاط الشركة)
  → قراءة الكلمات (تاب Keywords)  → تجميع
  → جلب Search Terms (Google Ads API)
  → تحضير المرشحين (Code):
        • اللي يطابق قاعدة → استبعاد مؤكد (source=rule)
        • الباقي → يتبعت لـ ChatGPT
  → ChatGPT تصنيف (HTTP → OpenAI): يرجّع لكل عبارة { exclude, reason, confidence }
  → دمج القرارات (Code): يجمّع قرارات القواعد + الـ AI
  → اقتراحات للمراجعة (Google Sheet → تاب Review)
```

### أعمدة تاب Review
`searchTerm | decision | source (rule/ai) | matchedWord | reason | confidence | suggestedNegative | matchType | campaignId | campaignName | clicks | cost | date`

### الاستيراد والتشغيل
1. في n8n: **Workflows → ⋯ → Import from File** واختار `workflow-ai-suggest.json`
   (أو Import from URL / لصق الـ JSON).
2. اعمل تابين جداد في نفس ملف Google Sheet: **`Keywords`** (فيه عمود `keyword`) و **`Review`**.
3. اربط الكريدنشيالز:
   - نودات **قراءة الكلمات** و **اقتراحات للمراجعة** → كريدنشيال Google Sheets.
   - نود **جلب Search Terms** → كريدنشيال **Google Ads OAuth2 API** (فيه adwords scope + Developer Token).
   - نود **ChatGPT تصنيف** → كريدنشيال **OpenAI**.
4. في نود **الإعدادات (Config)**: حط `customerId` و `loginCustomerId` و `developerToken`،
   وراجع `businessContext` (وصف نشاطك اللي الـ AI بيحكم على أساسه).
5. شغّل **Execute Workflow** يدوي، وراجع تاب **Review**.

### الموديل والتكلفة
- الموديل الافتراضي **`gpt-4o-mini`** (رخيص وسريع للتصنيف). غيّره من نود ChatGPT لو حبيت.
- الـ AI بيشوف بس العبارات اللي **القواعد مقفلتهاش**، فالتكلفة قليلة جداً.

### تفعيل الاستبعاد الأوتوماتيكي لاحقاً (لما تطمن)
تتضاف بعد نود "دمج القرارات":
- **IF**: `source == "rule"` **أو** `confidence >= 85` → يروح لنود **HTTP `campaignCriteria:mutate`**
  (زي اللي في `workflow.json`) عشان يضيف الـ negative فعلياً.
- الباقي (ثقة أقل) يفضل في Review للمراجعة اليدوية.
