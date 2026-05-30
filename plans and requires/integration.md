# integration-middleware-architecture.md

# Integration Middleware Architecture

## ERP / CRM PropTech System

---

# الهدف

بناء طبقة تكامل مرنة وقابلة للتوسع تربط نظام الـ ERP / CRM مع:

- بوابات الدفع
- منصات الحجز
- مدراء الحجز (Channel Managers)
- Webhooks
- الأنظمة الخارجية
- المنصات الإعلانية المستقبلية
- أنظمة التقارير
- مزامنة التقاويم (iCal)

بدون ربط الـ Business Logic مباشرة بالخدمات الخارجية.

---

# لماذا طبقة التكامل مهمة جدًا؟

في الأنظمة الحقيقية:

- كل مزود خارجي مختلف.
- كل API له قيوده.
- كل منصة لها Rate Limits.
- كل منصة تتغير باستمرار.

ولذلك:
لا يجب أن يعتمد الـ ERP مباشرة على APIs الخارجية داخل الـ Domain Logic.

---

# Architecture Principle

```txt id="pjxnpk"
ERP Core
   ↓
Integration Layer
   ↓
External Services
```

---

# ممنوع معماريًا

## ممنوع:

- استدعاء APIs الخارجية مباشرة من Controllers
- ربط Business Logic بـ Airbnb مباشرة
- كتابة Payment Logic داخل Sales Module
- التعامل مع Webhooks داخل الـ UI

---

# Integration Layer Responsibilities

طبقة التكامل مسؤولة عن:

- API Communication
- Data Mapping
- Retries
- Webhooks
- Queue Processing
- Rate Limiting
- Sync Jobs
- External Authentication
- Error Isolation

---

# التكاملات الأساسية الحالية

## Payment Gateway

### Paymob

التكامل مع:

- Payments
- Webhooks
- Transaction Verification
- Refunds
- Payment Status

---

# Booking Platforms Integration

## الهدف

مزامنة:

- الحجوزات
- التوافر
- التقاويم
- الأسعار

مع منصات الحجز.

---

# iCal Integration

## المكتبات

- ical-generator
- node-ical # استخدم لقراءة وتحليل ملفات iCal بشكل فوري (parsing + events)

OCR / Document Extraction Integration:

- Add a connector to call OCR providers (Google Cloud Vision / ML Kit) for structured extraction from images (national ID, water/electric bills).
- Flow: receive uploaded image -> enqueue OCR job -> provider returns OCR text -> parse into structured fields -> return results to caller and store metadata.
- Ensure rate-limiting, retries, and secure handling of PII in this connector.

- Pre-signed upload + OCR trigger flow: 1. الـ Frontend يضغط الصورة ويطلب من السيرفر رابط رفع مؤقت (pre-signed URL) من R2. 2. السيرفر يتحقق من الصلاحيات ويصدر رابط رفع موقّت ويرسله إلى المتصفح. 3. المتصفح يرفع الصورة مباشرة إلى R2 باستخدام الرابط الموقّت. 4. بعد نجاح الرفع، يبلّغ المتصفح السيرفر برابط الكائن في R2؛ عندها يقوم السيرفر بإدخال مهمة إلى الـ Queue لمعالجة OCR (يسحب الرابط من R2 ويرسله إلى Google Vision).

- بعد استلام نص الـ OCR: نفّذ تنظيفًا ثم استخدم Regex لاستخراج الحقول الدقيقة (مثلاً الرقم القومي 14 رقمًا). لا تعتمد فقط على الحقول المهيكلة التي قد تعطيها الخدمة — استخدم Keyword proximity ثم fallback إلى مطابقة \d{14}.

- لأرشفة قراءات العدادات: علّم الـ OCR بالبحث عن كلمات مفتاحية (مثل `الرصيد`, `جنيه`, `فاتورة`) ثم التقط الرقم الأقرب بعد الكلمة وسجّله كمؤشر `start_balance` أو `end_balance` مع ربطه بالـ `bookingId` المُستلم أثناء الرفع.

---

# لماذا iCal مهم؟

لأن:

- Airbnb
- Booking
- Google Calendar

يوفرون iCal مجانًا.

وبالتالي:
يمكن تنفيذ مزامنة قوية بدون APIs معقدة بالبداية.

---

# Smart Sync Strategy

## ممنوع عمل Cron Job عنيف

بدلًا من:

```txt id="6g1cmq"
Sync every minute for all properties
```

---

# استخدم Smart Cron

## الاستراتيجية

### الوضع الطبيعي

- تحديث كل 10 دقائق.

---

### عند فتح صفحة الدفع

إذا دخل عميل لصفحة الدفع لعقار:

يقوم الـ Backend مباشرة بـ:

- طلب iCal فورًا (باستخدام `node-ical` لقراءة وتحليل ودمج الأحداث)
- التحقق النهائي من التوفر
- منع Double Booking

---

## تحقق التوفر للمواعيد (عرض العقارات)

إذا كان الكالندر مخصصًا لمواعيد معاينة العقارات بين المشترين والمسوقين (Brokers):

- لحساب فترات التوفر (Availability Slots) استخدم `date-fns` للتعامل مع النطاقات والتداخلات الزمنية.
- عند جلب الأحداث من iCal، حوّلها إلى شرائح (slots) ثم استعمل دوال `date-fns` لاحتساب التعارضات والفراغات.

---

# لماذا هذا ممتاز؟

يوفر:

- API Requests أقل
- Compute أقل
- Free Tier عمر أطول
- دقة أعلى عند الدفع

---

# Example Flow

```txt id="h74bdb"
Client Opens Checkout
        ↓
Backend Fetches Latest iCal
        ↓
Validate Availability
        ↓
Continue Payment
```

---

# Webhooks System

## يستخدم لـ:

- Payment Notifications
- Booking Updates
- Reservation Changes
- External Events

---

# Webhook Rules

## المطلوب

- Signature Validation
- Retry Strategy
- Idempotency
- Logging
- Failure Recovery

---

# Message Queue

## الهدف

تشغيل العمليات الثقيلة خارج الـ Request Cycle.

---

# يستخدم لـ:

- Sending Emails
- iCal Sync
- Payment Verification
- Background Jobs
- Notifications
- Report Generation

---

# لماذا Message Queue مهمة؟

لأنها:

- تمنع بطء الـ API
- تمنع Timeout
- تعزل الأخطاء
- تحسن Scalability

---

# Queue Examples

```txt id="4u6d51"
User Books Property
      ↓
Create Booking
      ↓
Push Jobs To Queue
      ↓
Send Email
Sync Calendars
Generate Invoice
```

---

# Recommended Queue Tools

## البداية البسيطة

- BullMQ
- Redis

---

# مستقبلاً

- RabbitMQ
- Kafka

إذا كبر النظام.

---

# API Gateway

## الهدف

توحيد الدخول إلى التكاملات الخارجية.

---

# مسؤولياته

- Authentication
- Rate Limiting
- Logging
- Request Routing
- Monitoring

---

# Connectors Architecture

## يجب تصميم النظام ليقبل Connectors مستقبلية بسهولة.

---

# أمثلة Connectors

## Booking Platforms

- Airbnb
- Booking.com

---

# Channel Managers

- Guesty
- Hostaway
- Lodgify

---

# Future Integrations

- Advertising Platforms
- External Marketplaces
- SMS Providers
- WhatsApp Providers

---

# Connector Design Principle

كل Integration يجب عزله داخل Adapter مستقل.

---

# Example

```txt id="z4a9q8"
integrations/
│
├── paymob/
├── ical/
├── airbnb/
├── booking/
├── guesty/
└── webhooks/
```

---

# Adapter Pattern

## ممنوع:

```txt id="lbmknp"
Sales Module يعرف تفاصيل Paymob API
```

---

# الصحيح

```txt id="px50rk"
Sales Module
    ↓
Payment Service Interface
    ↓
Paymob Adapter
```

---

# Real-Time vs Batch Processing

# Real-Time APIs

تستخدم عندما تكون الدقة اللحظية مهمة.

---

# أمثلة

- Availability Check
- Payment Verification
- Inventory Sync

---

# Batch Processing

تستخدم عندما لا نحتاج تحديثًا لحظيًا.

---

# أمثلة

- Daily Reports
- Analytics
- Metrics Aggregation
- Nightly Sync Jobs

---

# قاعدة مهمة

لا تجعل كل شيء Real-Time.

لأن ذلك:

- مكلف
- يستهلك Compute
- يضغط على APIs الخارجية
- يستهلك Free Tier بسرعة

---

# Error Handling Strategy

## مطلوب

- Retry Logic
- Circuit Breakers
- Dead Letter Queue
- Logging
- Monitoring

---

# Security Rules

## ممنوع تخزين:

- Payment Secrets داخل Frontend
- API Keys داخل Client

---

# جميع الـ Secrets داخل Backend فقط

---

# Performance Principles

## المطلوب

- Future caching when external-read pressure or repeated provider lookups justify it
- Debouncing
- Smart Polling
- Queue Processing
- Lazy Synchronization

---

# Future Ready Architecture

النظام يجب أن يكون جاهزًا لاحقًا لـ:

- Multi Tenant
- Multiple Payment Gateways
- Multiple Booking Providers
- Marketplace APIs
- Mobile Apps
- Public APIs

---

# Recommended Research Topics

- Integration Middleware
- Adapter Pattern
- API Gateway Pattern
- Queue Based Architecture
- Event Driven Architecture
- BullMQ
- Webhooks Best Practices
- iCal Synchronization
- Rate Limiting
- Idempotency
- Circuit Breaker Pattern
- Distributed Systems Basics
