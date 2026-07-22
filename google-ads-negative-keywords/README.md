# ورك فلو Google Ads – الاستبعاد الأوتوماتيكي للكلمات السلبية

ورك فلو n8n بيدخل على حساب Google Ads **كل ٣ أيام**، يجيب الـ **Search Terms**،
يقارنها بقائمة كلمات ممنوعة إنت رافعها في **Google Sheet**، واللي يطابق يضيفه
**Negative Keyword** أوتوماتيك على مستوى الحملة.

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
