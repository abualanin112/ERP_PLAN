تنفيذ هذه الخصائص المتقدمة في بيئة `Express.js` بالاعتماد على سحابة `Cloudflare R2` يتم من خلال تقسيم المهام بين **لوحة تحكم السحابة (Infrastructure)** وبين **كود السيرفر (Application Logic)**.

السر هنا هو أنك لن تقوم ببرمجة خوارزميات معقدة من الصفر؛ بل ستستخدم واجهة برمجة تطبيقات (S3 API) لتوجيه الأوامر. إليك التفصيل التقني خطوة بخطوة:

### 1. الإدارة الفنية لتعدد الإصدارات (Versioning)

التحكم في الإصدارات لا يتم بإنشاء مجلدات جديدة في السيرفر، بل يتم على مستوى البنية التحتية.

- **كيف يتم تفعيله؟** تدخل إلى لوحة تحكم Cloudflare R2، تختار الـ Bucket الخاص بالمستندات (مثلاً `tut-secure-docs`)، وتقوم بتفعيل خيار **Object Versioning**.
- **الربط مع Express (البرمجة):**
  عندما يرفع الموظف نسخة معدلة من "عقد إيجار الوحدة 15" بنفس اسم الملف القديم `unit_15_contract.pdf`، يقوم R2 تلقائياً بحفظه كنسخة جديدة، ويُعيد لسيرفرك مُعرّفاً فريداً يُسمى `VersionId`.
  في قاعدة البيانات (Prisma/Mongoose)، تقوم بتحديث السجل ليحتفظ بهذا المُعرّف:

```javascript
// بعد نجاح الرفع، R2 يعيد كائن يحتوي على VersionId
const currentVersionId = r2Response.VersionId;

// تحديث قاعدة البيانات
await prisma.document.update({
  where: { id: docId },
  data: {
    versionId: currentVersionId,
    versionNumber: { increment: 1 },
  },
});
```

---

عندما تريد استدعاء النسخة القديمة لتسوية نزاع قانوني، تمرر الـ `VersionId` القديم في دالة `GetObjectCommand`، وسيجلب لك R2 النسخة السابقة.

---

### 2. سياسات الاحتفاظ والقفل (Lifecycle & Object Lock)

هذا الجزء يُدار بالكامل **بدون كتابة أي كود في Express**، مما يريح خادم DigitalOcean من أي عمليات جدولة (Cron Jobs) تستهلك الذاكرة.

- **الحذف التلقائي للمسودات (Lifecycle Rules):**
  من إعدادات الـ Bucket في لوحة تحكم R2، تذهب إلى **Lifecycle Rules** وتنشئ قاعدة (Rule):
  - _الشرط:_ أي ملف يبدأ مساره بـ `temp/` أو `drafts/`.
  - _الإجراء:_ `Delete Object` بعد `30 days`.
    سيقوم R2 بكنس هذه الملفات يومياً في صمت.

- **قفل المستندات القانونية (Object Lock - WORM):**
  (Write Once, Read Many). عند إنشاء الـ Bucket لأول مرة في R2، تفعل ميزة **Object Lock**. ثم تحدد فترة الاحتفاظ (مثلاً 5 سنوات).
  بمجرد رفع فاتورة الإيجار إلى هذا الـ Bucket، يتم قفل الملف فيزيائياً على سيرفرات Cloudflare. حتى لو تم اختراق سيرفر Express الخاص بك، أو قام مدير النظام بكتابة كود خاطئ يُرسل أمر `DeleteObject`، سيرفض R2 تنفيذ الأمر وسيعيد رسالة خطأ (Access Denied) حتى تنقضي الـ 5 سنوات.

---

### 3. التشفير (Encryption At Rest & In Transit)

هذا هو أسهل جزء هندسياً، لأن مزودي الخدمات السحابية الحديثة جعلوه افتراضياً:

- **في وضع السكون (At Rest):** لا تحتاج لكتابة كود تشفير بـ `crypto` في Node.js. Cloudflare R2 يقوم بتشفير جميع الملفات لحظة استقرارها على أقراصهم الصلبة باستخدام خوارزمية **AES-256** تلقائياً.
- **أثناء النقل (In Transit):** يتم فرض استخدام بروتوكول **TLS 1.2/1.3**. عندما يقوم كود Express الخاص بك أو متصفح العميل بالتخاطب مع R2، فإن نقطة الاتصال (Endpoint) تكون إجبارياً عبر `https://`. أي محاولة للاتصال عبر `http` غير المشفر يتم رفضها فوراً.

---

### 4. التنفيذ البرمجي للروابط الموقعة مسبقاً (Pre-signed URLs)

هذا هو الدرع الأمني الفعلي داخل تطبيق Express. نستخدم حزمة `@aws-sdk/s3-request-presigner` لتوليد توقيع تشفيري (Cryptographic Signature) يدمج بيانات اعتمادك (Secret Key) مع وقت الصلاحية.

إليك الكود التقني لكيفية توليد رابط يسمح بعرض البطاقة الشخصية لعميل لمدة 5 دقائق فقط:

```js
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

// 1. إعداد الاتصال السحابي باستخدام مفاتيح توت (تُحفظ في .env)
const r2Client = new S3Client({
  region: "auto",
  endpoint: process.env.R2_ENDPOINT, // https://<ACCOUNT_ID>.r2.cloudflarestorage.com
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY,
    secretAccessKey: process.env.R2_SECRET_KEY,
  },
});

// 2. دالة توليد الرابط الآمن داخل Express
async function generateSecureViewUrl(fileKey) {
  // تجهيز أمر جلب الملف
  const command = new GetObjectCommand({
    Bucket: "tut-secure-docs", // مساحة التخزين المغلقة
    Key: fileKey, // مسار الملف، مثال: "ids/2026/client_99.jpg"
  });

  // توليد التوقيع: 300 ثانية = 5 دقائق
  // الدالة لا تتصل بالإنترنت، بل تقوم بعملية تشفير رياضية داخل سيرفرك
  const secureUrl = await getSignedUrl(r2Client, command, { expiresIn: 300 });

  return secureUrl;
}

// 3. مسار Express (Endpoint)
app.get("/api/documents/:docId", requireAdminAuth, async (req, res) => {
  try {
    // استخراج مسار الملف من الداتا بيز (Neon)
    const docRecord = await prisma.document.findUnique({
      where: { id: req.params.docId },
    });

    // توليد الرابط
    const signedUrl = await generateSecureViewUrl(docRecord.fileKey);

    // إرسال الرابط للواجهة الأمامية
    res.json({ url: signedUrl });
  } catch (error) {
    res.status(500).json({ error: "فشل في تأمين المستند" });
  }
});
```

الرابط الناتج سيكون طويلاً جداً ويحتوي على متغيرات مثل `X-Amz-Signature` و `X-Amz-Expires`. إذا حاول أي شخص تغيير حرف واحد في الرابط، أو انتهت الـ 300 ثانية، سيرفض خادم R2 عرض الصورة فوراً.
