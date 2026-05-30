# Application Server Architecture Guide

## Modular Monolith Architecture

### Express.js + Prisma + PostgreSQL

Note: This workspace uses an `Express` backend with a `React` frontend; keep the API server stateless and let React handle UI and calendar interactions (iCal flows use the Integration Layer).

OCR / Document Extraction Service:

- Add a small service in the Integration Layer to handle OCR requests (national ID, water/electric bills). Prefer using Google Cloud Vision API (or ML Kit) as the OCR engine — this can be called from the backend to extract structured fields.
- Flow: frontend compresses image -> API issues a pre-signed R2 upload URL after auth checks -> browser uploads directly to R2 -> API stores metadata and enqueues OCR job -> OCR service calls Google Vision -> parse fields (name, national id, invoice number, amount, due date) -> persist metadata in Neon.
- Image processing: `sharp` محذوف من المعمارية؛ جميع عمليات الضغط، تغيير المقاسات، والعلامات المائية تُجرى داخل متصفح العميل (React + Canvas). دور الخادم يقتصر على إصدار روابط الرفع الموقّتة، التحقق من الأذونات، إدخال مهام OCR/الميتا في الصف، وتخزين المراجع.
- Security: redact or encrypt sensitive fields at rest, log access, and enforce retention policies.

### الربط التقني: Integration Layer (Prisma & Connection Pooling)

- **استخدم Prisma كـ ORM:** استخدم Prisma كجسر آمن بين `Express` وNeon/Postgres للعمليات اليومية (CRUD). Prisma يحسّن تجربة المطور، يوفّر Type-safety، ويحمي من SQL injection.

- **تفعيل Connection Pooling:** Neon يوفّر نوعين من روابط الاتصال — Direct وPooler. لا تستخدم الـ Direct Connection في بيئات الإنتاج لأنّه يستهلك اتصالات بسرعة. ضع رابط الـ Pooler في `.env` (مثلاً `DATABASE_URL`) ليعمل السيرفر عبر Pooler أو `pgBouncer`.

- **مثال إعداد بيئة (.env):**

```env
DATABASE_URL=postgresql://pooler-user:xxxx@ep-pooler.cluster.neon.tech/yourdb?sslmode=require
```

- **تهيئة Prisma:** Prisma يقرأ `DATABASE_URL` تلقائيًا. في أغلب الحالات يكفي:

```js
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();
```

- **أفضل الممارسات التشغيلية:**
  - اجعل المعاملات قصيرة لتقليل القفل واستهلاك الاتصالات.
  - استخدم `SELECT ... FOR UPDATE` داخل معاملات عند الحاجة لتجنّب الكتابة المتزامنة على نفس الصف.
  - راقب استخدام الاتصالات في Neon dashboard واضبط إعدادات الـ pooler عندما يلزم.

## نسخ احتياطي ونسخ آمنة (Backup & Safety)

حتى مع خاصيات Neon، من الأفضل وجود خطة نسخ احتياطية مستقلة. اقتراح عملي: جدولة نسخة يومية باستخدام `pg_dump` ورفعها إلى Cloudflare R2.

- **فوائد:** سيضمن ذلك استعادة سريعة عند فشل مزوّد الخدمة أو تغيّر الإعدادات، ويمنحك تحكماً كاملاً في فترة الاحتفاظ.

- **ملاحظة تشغيلية:** شغّل هذا في خادم دائم (مثلاً Droplet على DigitalOcean) أو في Job runner مخصّص — لا تشغله داخل بيئات lambda قصيرة العمر بدون إعداد صحيح للـ filesystem.

- **مثال Node.js (مبسّط):**

```js
// تثبيت الحزم: npm i node-cron @aws-sdk/client-s3
import cron from "node-cron";
import { exec } from "node:child_process";
import fs from "node:fs";
import path from "node:path";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({
  endpoint: process.env.R2_ENDPOINT,
  region: "auto",
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY,
    secretAccessKey: process.env.R2_SECRET_KEY,
  },
});

cron.schedule("0 3 * * *", async () => {
  const filename = `backup-${new Date().toISOString().slice(0, 10)}.sql.gz`;
  const tmpPath = path.join("/tmp", filename);

  // استخدم رابط نسخ احتياطي مباشر/قراءة فقط، واترك DATABASE_URL للـ Pooler الخاص بالتطبيق.
  const dumpCmd = `pg_dump "${process.env.BACKUP_DATABASE_URL}" | gzip > ${tmpPath}`;

  exec(dumpCmd, async (err) => {
    if (err) return console.error("pg_dump failed", err);
    const fileStream = fs.createReadStream(tmpPath);
    await s3.send(
      new PutObjectCommand({
        Bucket: process.env.BACKUP_BUCKET,
        Key: filename,
        Body: fileStream,
      }),
    );
    fs.unlinkSync(tmpPath);
    console.log("Backup uploaded to R2:", filename);
  });
});
```

- **أمن:** خزّن اسم الـ Bucket وبيانات الوصول في متغيرات بيئة آمنة، وحصر صلاحيات مفاتيح R2 على عملية الرفع فقط.

### المعمارية الهجينة (محدودة): استثناء `Multer` للـ Bulk Import

المبدأ: مسار رفع الصور اليومي يعتمد كليًا على معالجة العميل ورفع مباشر إلى R2. الخادم لا يستخدم `Sharp` إطلاقًا.

الاستثناء العملي الوحيد المسموح به هو:

1. **استيراد دفعات CSV / Excel (Bulk Import)**
   - استخدم `Multer.memoryStorage()` لمسارات استيراد إدارية فقط لقراءة الملفات في الذاكرة ومعالجة الصفوف دفعة واحدة، ثم أفرغ الذاكرة.

لا توجد استثناءات لمعالجة الصور اليومية على الخادم — جميع عمليات الضغط، تغيير المقاسات، والعلامات المائية تُجرى في المتصفح قبل الرفع.

### التعامل برمجياً مع البطاقات والكروت (في Express.js)

- بعد أن يُرجع موّرد الـ OCR (Google Vision) النص الكامل كسلسلة واحدة، لا تعتمد كليًا على الحقول المهيكلة التي قد تُعطيها الخدمة — استخدم خطوات متسلسلة تعتمد على التنظيف والـ Regex لاستخراج الرقم القومي بدقة.
- الخوارزمية الموصى بها:
  1.  خفّف النص من فراغات زائدة: `const normalized = ocrText.replace(/\s+/g, ' ');`
  2.  ابحث عن كلمات مفتاحية مجاورة مثل `الرقم القومي` ثم التقط الـ 14 رقماً التي تليها مباشرة.
  3.  إذا فشل البحث بالكلمة المفتاحية، استخدم مطابقة احتياطية للعثور على أي مجموعة مكوّنة من 14 رقمًا.

مثال (Node/Express):

```js
// ocrText: السلسلة التي أرجعها Google Vision
const text = ocrText.replace(/\s+/g, " ");
let nationalId = null;

// حاول الاستخراج قرب الكلمات المفتاحية أولاً
const keywordMatch = text.match(/(?:الرقم\s*القومي|الرقم)\D*(\d{14})/i);
if (keywordMatch) {
  nationalId = keywordMatch[1];
}

// احتياطي: أي سلسلة مكونة من 14 رقمًا
if (!nationalId) {
  const fallback = text.match(/\d{14}/);
  nationalId = fallback ? fallback[0] : null;
}

// nationalId الآن إمّا رقم قومي صالح أو null
```

- قراءة فواتير المرافق (كروت الكهرباء/المياه): علّم الـ OCR بالبحث عن كلمات مفتاحية مثل `الرصيد`, `فاتورة`, `جنيه`، ثم التقط الرقم الذي يليها لتسجيل `start_balance` أو `end_balance`.
- ربط بالـ Booking: عند `check-in` أو `check-out`، استقبل `bookingId` مع عملية الرفع أو الإخطار، وحفظ مرجع الصورة والحقول المستخرجة في صف مرتبط بـ `booking_id` لقابلية المراجعة والمطابقة المحاسبية.
- ضع قواعد أرشفة: احفظ الصورة الأصلية تحت مسار محمي في R2 (مثلاً `private/ids/`) واحتفظ بنسخة محسّنة للاستخدام اليومي، وسجل السجل الكامل للـ OCR والناتج بعد الـ Regex للتمكين من عمليات التدقيق لاحقًا.

## فيرشيننج النصوص والعقود (Server-side responsibilities)

في جانب السيرفر، يجب أن يضمن التطبيق سلوكًا موحّدًا عند تحديث نصوص العقود أو قوالب الشروط:

- **لا تمسح القديم — أنشئ نسخة جديدة:** عند تعديل نص عقد، أنشئ سجلًا جديدًا في `contract_history` ثم حدّث مرجع النسخة الحالية في `contracts` داخل نفس المعاملة (transaction).

- **ربط القبول بالحجز:** عندما يوافق الضيف على نص معين أثناء `check-in`، خزّن `contract_history.id` و`version` داخل صف `bookings` (مثال: `booking.contract_snapshot_id`, `booking.contract_version`) لتضمن إمكانية استرجاع النص الصحيح لاحقًا.

- **مثال Express + Prisma (موجز):**

```js
import { createHash } from "node:crypto";

// createContractVersion: ينشئ نسخة جديدة ويحفظ المرجع في جدول العقود
async function createContractVersion(contractId, contentJson, userId, reason) {
  return await prisma.$transaction(async (tx) => {
    // اقرأ الإصدار الحالي بطريقة آمنة (SELECT ... FOR UPDATE) في حالة التزامن العالي
    const contract = await tx.contract.findUnique({
      where: { id: contractId },
    });
    const newVersion = (contract.currentVersion || 1) + 1;

    const version = await tx.contractHistory.create({
      data: {
        contractId,
        version: newVersion,
        content: contentJson,
        changeReason: reason,
        changedBy: userId,
        changeHash: createHash("sha256")
          .update(JSON.stringify(contentJson))
          .digest("hex"),
      },
    });

    await tx.contract.update({
      where: { id: contractId },
      data: { currentVersion: newVersion, currentSnapshotId: version.id },
    });

    return version;
  });
}
```

- **التحقق والامتثال:** بمجرد ربط نسخة بعقد حجز، اجعل هذه النسخة غير قابلة للتعديل بسهولة (read-only) واحتفظ بسجلات من الذي قام بإنشائها ولماذا.

- **تكامل مع الـ Audit Log:** انشر حدثًا (event) عبر الصفوف (queue) عند إنشاء نسخة جديدة لاحتساب السجلات ضمن `audit_logs` الخارجية ومراقبة التغيرات.

> This document defines the recommended Application Server architecture for a modern PropTech ERP/CRM platform using a pragmatic modular monolith approach.

---

# Core Philosophy

The system should be:

- modular
- scalable
- maintainable
- easy to debug
- easy to extend

The architecture SHOULD:

- separate business domains
- isolate responsibilities
- avoid tightly coupled code
- support async processing

The architecture SHOULD NOT:

- over-engineer early
- use distributed microservices too soon
- introduce complex event systems prematurely

---

# Recommended Architecture Style

# Modular Monolith

The application runs as:

- one deployable application
- one backend codebase
- one PostgreSQL database

But internally divided into:

- business modules
- services
- infrastructure layers

---

# High-Level Architecture

```txt id="m1"
Frontend
   ↓
Express API Layer
   ↓
Application Services
   ↓
Domain Modules
   ↓
Prisma ORM
   ↓
PostgreSQL
```

Additional Infrastructure:

```txt id="m2"
Redis
BullMQ
Background Workers
```

---

# Recommended Project Structure

```txt id="m3"
src/
 ├── app/
 │
 ├── modules/
 │    ├── bookings/
 │    │     ├── booking.controller.js
 │    │     ├── booking.service.js
 │    │     ├── booking.repository.js
 │    │     ├── booking.validator.js
 │    │     ├── booking.serializer.js
 │    │     └── index.js
 │    │
 │    ├── properties/
 │    ├── payments/
 │    ├── owners/
 │    ├── finance/
 │    ├── notifications/
 │    ├── analytics/
 │
 ├── workers/
 │    ├── email.worker.js
 │    ├── report.worker.js
 │    ├── invoice.worker.js
 │
 ├── queues/
 │    ├── email.queue.js
 │    ├── report.queue.js
 │
 ├── infrastructure/
 │    ├── prisma.js
 │    ├── redis.js
 │    ├── logger.js
 │
 ├── shared/
 │    ├── errors/
 │    ├── utils/
 │    ├── middleware/
 │
 ├── config/
 │
 ├── index.js
```

---

# Layer Responsibilities

# 1. Controllers

Controllers should:

- receive HTTP requests
- validate input
- call services
- return responses

Controllers SHOULD NOT:

- contain business logic
- perform heavy database operations
- coordinate workflows

---

# Good Example

```js id="m4"
export async function createBooking(req, res) {
  const booking = await bookingService.create(req.body);

  return res.json(booking);
}
```

---

# Bad Example

```txt id="m5"
Controller with:
- pricing logic
- validation rules
- payment handling
- invoice generation
- analytics updates
```

---

# 2. Application Services

Application Services coordinate workflows.

Examples:

- BookingService
- PaymentService
- RevenueService
- AvailabilityService

Services handle:

- business workflows
- orchestration
- validation coordination
- transactions
- queue dispatching

---

# Example Booking Workflow

```txt id="m6"
Validate availability
   ↓
Calculate pricing
   ↓
Create booking
   ↓
Reserve dates
   ↓
Generate invoice
   ↓
Queue confirmation email
```

---

# Example Service

```js id="m7"
async function createBooking(data) {
  await validateAvailability(data);

  const booking = await prisma.booking.create({
    data,
  });

  await invoiceService.generate(booking);

  await emailQueue.add("booking-confirmation", {
    bookingId: booking.id,
  });

  return booking;
}
```

---

# 3. Repository Layer

Repositories isolate database access.

Responsibilities:

- Prisma queries
- Raw SQL queries
- database transactions
- query optimization

---

# Repository Example

```js id="m8"
async function findAvailableProperties(filters) {
  return prisma.property.findMany({
    where: {
      city: filters.city,
    },
  });
}
```

---

# 4. Prisma ORM Usage

Use Prisma for:

- CRUD operations
- relations
- transactional writes
- type safety

DO NOT use Prisma for:

- heavy analytics
- large aggregations
- complex reporting queries

---

# Raw SQL Strategy

Use Raw SQL for:

- analytics
- dashboards
- financial reports
- aggregation queries

Example:

```js id="m9"
await prisma.$queryRaw`
SELECT
   property_id,
   COUNT(*) bookings,
   SUM(total_price) revenue
FROM bookings
GROUP BY property_id
`;
```

---

# Database Responsibilities

PostgreSQL handles:

- transactions
- reporting
- aggregations
- indexing
- partitioning
- materialized views

The application server handles:

- workflows
- orchestration
- business rules

---

# Stateless Architecture

The API layer MUST remain stateless.

DO NOT:

- store sessions in memory
- keep workflow state in local process memory

Use:

- Redis
- JWT authentication

---

# Correct Architecture

```txt id="m10"
Load Balancer
   ↓
Multiple API Servers
   ↓
Shared Redis
```

---

# Validation Strategy

Validation exists in multiple layers.

---

# API Validation

Validate:

- request body
- query params
- DTO schemas

Recommended:

- Zod
- Joi
- class-validator

---

# Business Validation

Validate:

- booking overlap
- pricing rules
- owner restrictions
- refund eligibility

---

# Database Validation

Use:

- foreign keys
- unique constraints
- indexes

---

# Queue Architecture

Use queues for:

- emails
- PDF generation
- reports
- notifications
- integrations
- image processing

Recommended:

- BullMQ
- Redis

---

# Queue Flow

```txt id="m11"
HTTP Request
   ↓
Application Service
   ↓
Queue Job
   ↓
Worker
   ↓
Background Processing
```

---

# Email Queue Example

## Queue

```js id="m12"
await emailQueue.add("booking-email", {
  bookingId,
});
```

---

# Worker

```js id="m13"
worker.process(async (job) => {
  await sendBookingEmail(job.data);
});
```

---

# Background Workers

Workers execute:

- long-running tasks
- heavy processing
- scheduled jobs

Examples:

- reports
- invoice PDFs
- archive jobs
- analytics refresh
- owner statements

Never execute these inside HTTP requests.

---

# Transaction Strategy

Use transactions for:

- booking creation
- payment processing
- owner payouts
- invoice creation

Example:

```js id="m14"
await prisma.$transaction(async (tx) => {

   const booking =
      await tx.booking.create(...);

   await tx.invoice.create(...);

});
```

---

# Future Caching Strategy

Read caching is a future performance phase, not a default first-release requirement. Start with PostgreSQL indexes, optimized queries, materialized/read models, and monitoring. Add Redis read cache only when metrics show repeated read pressure.

Future Redis read cache can be used for:

- property search cache
- dashboard cache
- frequently accessed settings

Avoid caching, even later:

- highly volatile transactional state

---

# Error Handling

Use centralized:

- error middleware
- structured exceptions
- logging

Never expose:

- stack traces
- raw database errors

---

# Logging Strategy

Log:

- requests
- errors
- worker failures
- queue failures
- performance metrics

Recommended:

- structured JSON logs

---

# Security Strategy

Use:

- JWT authentication
- RBAC authorization
- audit logging

Critical operations should create audit logs.

Examples:

- booking modification
- payout approval
- pricing changes

---

# API Design Principles

Use:

- REST APIs

Recommended:

- pagination
- API versioning
- standardized responses

Avoid:

- giant generic endpoints
- exposing internal database structures

---

# Monitoring

Monitor:

- API latency
- queue latency
- worker failures
- slow database queries
- Redis performance if Redis is actively used

Recommended Tools:

- Prometheus
- Grafana
- pg_stat_statements

---

# Scalability Principles

Scale using:

- multiple API servers
- distributed workers
- future Redis cache when metrics justify it
- PostgreSQL optimization

Avoid:

- sticky sessions
- local process state
- giant monolithic services

---

# Example Real PropTech Workflow

# Create Booking

## Step 1

User submits booking request

↓

## Step 2

BookingService validates:

- availability
- pricing
- booking rules

↓

## Step 3

Create booking transaction

↓

## Step 4

Reserve availability

↓

## Step 5

Generate invoice

↓

## Step 6

Queue confirmation email

↓

## Step 7

Worker sends email asynchronously

---

# Coding Standards

# Service Rules

Services SHOULD:

- focus on one business capability
- remain small
- coordinate workflows

Services SHOULD NOT:

- contain HTTP logic
- directly manipulate response objects

---

# Controller Rules

Controllers SHOULD:

- remain thin
- call services only

Controllers SHOULD NOT:

- contain business logic

---

# Database Rules

Use:

- indexes
- transactions
- normalized schema

Avoid:

- huge unbounded tables
- ORM-only architecture
- large live analytics queries

---

# Queue Rules

Use queues for:

- slow tasks
- async tasks
- retryable tasks

Avoid:

- heavy synchronous processing

---

# Anti-Patterns

DO NOT:

- place all logic in controllers
- use giant shared utility services
- build microservices too early
- introduce Kafka too early
- over-engineer workflows
- run reports inside HTTP requests

---

# Recommended Stack

API Layer:

- Express.js

ORM:

- Prisma

Database:

- PostgreSQL

Future Cache:

- Redis

Queues:

- BullMQ

Workers:

- Node.js workers

Deployment:

- Docker

Gateway:

- Nginx

Monitoring:

- Prometheus
- Grafana

---

# Engineering Philosophy

The application server is:

- a workflow coordinator
- a business orchestration layer
- a security boundary

It is NOT:

- a giant CRUD controller
- a reporting engine
- a distributed event system

Build simple first.
Scale architecture gradually as complexity grows.
