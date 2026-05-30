# PropTech Database Architecture Guide

## PostgreSQL + Prisma + Express/NestJS

> This document defines the database architecture, scaling strategy, performance patterns, and operational best practices for a modern PropTech ERP/CRM platform.

> Neon يستطيع تشغيل شركة واحدة لمدة شهرين مجانًا — بل ربما أكثر بكثير — إذا كان الاستخدام طبيعيًا ولم تبنِ النظام بطريقة تستهلك الموارد بشكل سيئ.

---

# Core Database Philosophy

The database is NOT:

- just a storage engine
- or a simple CRUD layer

In enterprise systems, PostgreSQL is responsible for:

- data integrity
- transactional consistency
- financial accuracy
- analytics
- reporting
- scalability
- heavy data processing

---

# Database Stack

## Primary Database

- PostgreSQL

Recommended Hosted Option: Neon (Postgres-compatible) — good fit for Postgres workloads and Prisma metadata.

## ORM Layer

- Prisma ORM

### Integration Layer & Connection Pooling

- **Prisma كجسر بين Express وPostgres:** استخدم Prisma كـ ORM للعمليات اليومية (CRUD). ضع `DATABASE_URL` في `.env` ودع Prisma يتصل بقاعدة Neon عبر الرابط الموجود في لوحة التحكم.

- **استخدم Pooler وليس الـ Direct Connection في الإنتاج:** Neon يقدّم رابطين: `Direct` و`Pooler` (pgBouncer-like). الرابط الخاص بالـ Pooler يجب أن يكون المتغيّر في `.env` للسيرفر (`DATABASE_URL`) لأنّه يدير الاتصالات بشكل فعّال ويمنع استنزاف الـ connections من الخادم.

- **مثال `.env` (نموذجي):**

```env
DATABASE_URL=postgresql://pooler-user:XXXXXXXX@ep-pooler.cluster.neon.tech/yourdb?sslmode=require
```

- **تهيئة `schema.prisma`:**

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

- **ملاحظات تشغيلية:**
  - اجعل المعاملات قصيرة واستخدم `SELECT ... FOR UPDATE` عند الحاجة لحماية الصفوف المتزامنة.
  - راقب عدد الاتصالات في Neon dashboard واضبط إعدادات Pooler (max client connections) حسب الحمل.
  - في بيئات Serverless أو Auto-scaling، اعتمد دومًا على Pooler لتجنّب انهيار قاعدة البيانات بسبب انفجار الاتصالات.

## Future Cache Layer

- Redis read caching is deferred until performance metrics justify it.

## Queue Infrastructure

- BullMQ + Redis

## Background Processing

- Node.js Workers

---

# Database Responsibilities

PostgreSQL handles:

- ACID transactions
- financial operations
- aggregations
- analytics
- reporting
- partitioning
- materialized views
- indexing
- data integrity
- stored procedures

---

# Prisma Responsibilities

Prisma is used for:

- CRUD operations
- relations
- type-safe access
- transactional writes
- developer productivity

Prisma SHOULD NOT be used for:

- large analytics queries
- heavy reporting
- complex aggregations
- financial rollups
- massive bulk updates

---

# Database Design Principles

# 1. Normalize Core Transactional Data

Use normalized schema for:

- users
- bookings
- properties
- owners
- payments
- invoices

Example:

```txt
users
properties
bookings
booking_items
payments
owners
```

Avoid:

- duplicated data
- giant JSON blobs
- repeated customer data

---

# 2. Use Foreign Keys

Always enforce relational integrity.

Example:

```sql
FOREIGN KEY (property_id)
REFERENCES properties(id)
```

Never rely only on application validation.

---

# 3. Use Proper Indexing

Indexes are mandatory for:

- foreign keys
- search fields
- filtering fields
- sorting fields

Examples:

- property_id
- owner_id
- booking_date
- created_at
- payment_status

---

# Example Index

```sql
CREATE INDEX idx_bookings_property
ON bookings(property_id);
```

---

# Indexing Best Practices

DO:

- index frequently queried columns
- index JOIN columns
- index date filters
- benchmark queries

DO NOT:

- index every column
- create unnecessary indexes
- ignore index maintenance

---

# Query Optimization

Use:

- EXPLAIN ANALYZE
- query benchmarking
- pg_stat_statements

Monitor:

- slow queries
- sequential scans
- expensive joins
- large sorts

---

# Raw SQL Strategy

Use Raw SQL for:

- dashboards
- reports
- aggregations
- analytics
- financial calculations

Example:

```sql
SELECT
  property_id,
  COUNT(*) total_bookings,
  SUM(total_price) revenue
FROM bookings
GROUP BY property_id;
```

---

# Prisma + Raw SQL Pattern

Use:

- Prisma for CRUD
- Raw SQL for heavy operations

Example:

```js
await prisma.$queryRaw`
SELECT *
FROM bookings
WHERE created_at >= NOW() - INTERVAL '30 days'
`;
```

---

# Stored Procedures

Use Stored Procedures ONLY for:

- financial closing
- commission calculations
- owner payouts
- bulk transactional processing
- accounting operations

Example:

```sql
CREATE OR REPLACE PROCEDURE close_monthly_revenue(period_id INT)
```

---

# Stored Procedure Best Practices

DO:

- keep procedures focused
- add audit logs
- make operations idempotent
- wrap in transactions

DO NOT:

- move all business logic into PostgreSQL
- create huge monolithic procedures

---

# Transaction Strategy

Use transactions for:

- booking creation
- payment processing
- owner payout calculations
- financial operations

Example:

```js
await prisma.$transaction(async (tx) => {
  // transactional operations
});
```

---

# Large Table Strategy

Tables expected to grow heavily:

- bookings
- payments
- audit_logs
- notifications
- availability_history
- financial_transactions

These tables must eventually use:

- partitioning
- archiving

---

## Versioning النصوص والعقود (Text Versioning & Audit Trail)

نصوص العقود، قوالب الشروط، ولقطات النصوص التي يوافق عليها العملاء تحتاج إلى فيرشيننج داخلي داخل قاعدة البيانات — ليس فقط على مستوى ملفات الـ PDF — للأسباب التالية: احتياج لمسار تدقيق (audit trail)، استدعاء اللقطة الدقيقة عند النزاعات، والحفاظ على سلامة السجلات المحاسبية عبر المواسم.

نمط التنفيذ المقترح (Patterns):

- **جداول تدقيق (Audit Tables)**: لا تقم بالـ UPDATE الذي يمحو القديم، بل أدرج نسخة جديدة في جدول `contract_history` مع `version_number`، `content` (JSONB/TEXT)، `changed_by`, `changed_at`, `change_reason`, و`change_hash` (SHA-256) لتسهيل التحقق من عدم التلاعب.

- **حقل الإصدار في الجدول الرئيسي**: احتفظ في جدول `contracts` بحقل `current_version` و`current_snapshot_id` ليشير إلى آخر نسخة فعّالة، لتسهيل الاستعلام السريع للنسخة الحالية.

- **ربط القبول مع الحجز (Booking acceptance snapshot)**: عند قبول العميل لشروط/عقد أثناء `check-in`، خزّن `contract_history.id` أو `(contract_id, version)` داخل صف `bookings` ليضمن إمكانية استرجاع النص الدقيق الذي وافق عليه هذا العميل.

- **خوارزمية الحفظ (Transactional write)**: استخدم معاملات (transactions) عند إنشاء نسخة جديدة: أولًا إنشـر سجلًا في `contract_history` ثم حدّث `contracts.current_version/current_snapshot_id` داخل نفس المعاملة لتفادي حالات التزامن.

- **التحقق من عدم التلاعب (Tamper detection)**: خزّن هاش SHA-256 للمحتوى (`change_hash`) عند الإنشاء — يمكن التحقق لاحقًا من أن النص لم يتغيّر.

أمثلة عملية (Prisma / PostgreSQL):

Prisma (نماذج مقتصرة):

```prisma
model Contract {
  id                BigInt  @id @default(autoincrement())
  propertyId        BigInt
  currentVersion    Int     @default(1)
  currentSnapshotId BigInt?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  history           ContractHistory[]
}

model ContractHistory {
  id               BigInt   @id @default(autoincrement())
  contractId       BigInt
  version          Int
  content          Json
  changeReason     String?
  changedBy        BigInt?
  changedAt        DateTime @default(now())
  changeHash       String
  isSigned         Boolean  @default(false)
  acceptedBookingId BigInt?
}
```

SQL (إدراج نسخة جديدة داخل معاملة):

```sql
-- يتطلب امتداد pgcrypto لـ digest()
BEGIN;
INSERT INTO contract_history (contract_id, version, content, change_reason, changed_by, change_hash, changed_at)
VALUES (123, 2, '{"text":"..."}'::jsonb, 'تحديث بنود الإلغاء', 42, encode(digest('{"text":"..."}','sha256'),'hex'), now())
RETURNING id;

-- افترض أن الـ id العائد هو 456
UPDATE contracts
SET current_version = 2, current_snapshot_id = 456
WHERE id = 123;
COMMIT;
```

اختيار بديل: **Trigger-level audit** — يمكنك تنفيذ Trigger في PostgreSQL ينسخ النسخ القديمة إلى `contract_history` تلقائيًا قبل أي UPDATE، لكن يظل من الأفضل التحكم بالكتابة عبر التطبيق عندما تحتاج إلى bind لبيانات المستخدم (who/why).

استعلامات استرجاع سريعة:

- الحصول على النص الذي وافق عليه حجز معيّن:

```sql
SELECT ch.content
FROM bookings b
JOIN contract_history ch ON ch.id = b.contract_snapshot_id
WHERE b.id = $1;
```

اعتبارات عملية وأمنية:

- اجعل الـ `contract_history` قابلاً للقراءة عبر واجهات API ولكن ممنوع الحذف العادي — استخدم سياسات احتفاظ/أرشفة واضحة.
- احتفظ بعنصر `accepted_by` و`accepted_at` في صف الحجز (Booking) لربط القبول الرسمي بنسخة النص.

---

# Partitioning Strategy

# Purpose

Partitioning improves:

- large table management
- query performance
- index efficiency
- maintenance
- archiving

---

# Recommended Partitioning Type

Use:

- RANGE partitioning by date

Examples:

- created_at
- booking_date
- payment_date

---

# Partitioned Table Example

```sql
CREATE TABLE bookings (
    id BIGSERIAL,
    property_id BIGINT,
    customer_id BIGINT,
    total_price NUMERIC,
    created_at TIMESTAMP NOT NULL
)
PARTITION BY RANGE (created_at);
```

---

# Monthly Partition Example

```sql
CREATE TABLE bookings_2026_01
PARTITION OF bookings
FOR VALUES FROM ('2026-01-01')
TO ('2026-02-01');
```

---

# Partitioning Best Practices

DO:

- partition only large tables
- use monthly or yearly partitions
- create indexes per partition
- automate partition creation

DO NOT:

- over-partition
- create thousands of partitions
- partition small tables

---

# Partition Automation

Create future partitions automatically using:

- cron jobs
- migrations
- background workers

Example:

- create next month partition automatically

---

# Partition Pruning

PostgreSQL automatically scans only relevant partitions.

Example:

```sql
SELECT *
FROM bookings
WHERE created_at >= '2026-01-01'
```

Only matching partitions are scanned.

---

# Archiving Strategy

# Purpose

Archive old inactive data to:

- reduce production database size
- improve performance
- reduce backup costs

---

# Hot vs Cold Data

## Hot Data

Frequently accessed:

- current year
- recent bookings
- active payments

## Cold Data

Rarely accessed:

- old bookings
- old invoices
- historical logs

---

# Archive Strategy

Keep:

- last 1-2 years active

Move older data to:

- archive database
- cold storage
- compressed exports

---

# Archive Workflow

```txt
Monthly Archive Job
    ↓
Find old partitions
    ↓
Detach partition
    ↓
Move to archive database
    ↓
Compress backup
    ↓
Delete from production
```

---

# Detaching Partitions

Example:

```sql
ALTER TABLE bookings
DETACH PARTITION bookings_2023;
```

---

# Archive Infrastructure

Recommended:

- separate archive PostgreSQL database
- object storage backups
- compressed snapshots

---

# Future Cache Layer

Read caching should not be part of the initial database plan. First use indexing, query optimization, materialized views, and read models. Add Redis read cache later when query latency or repeated read load proves the need.

Future Redis cache can be used for:

- dashboard caching
- property search caching
- settings
- frequently accessed data

DO NOT cache:

- sensitive financial transactions
- rapidly changing booking state

---

# Materialized Views

Use Materialized Views for:

- KPIs
- analytics dashboards
- occupancy metrics
- revenue summaries

Example:

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
  DATE_TRUNC('month', created_at) month,
  SUM(total_price) revenue
FROM bookings
GROUP BY month;
```

Refresh:

```sql
REFRESH MATERIALIZED VIEW monthly_revenue;
```

---

# Materialized View Best Practices

DO:

- refresh periodically
- combine with Redis cache later only when metrics justify it
- use for expensive aggregations

DO NOT:

- refresh too frequently
- use for transactional reads

---

# Read Models

Read Models are optimized read-only structures.

Use for:

- dashboards
- analytics
- reporting
- KPIs

Example:

```txt
owner_dashboard_stats
property_analytics
monthly_revenue_summary
```

---

# Queue + Worker Database Strategy

Heavy database operations must run in workers.

Use workers for:

- reports
- exports
- financial calculations
- archive jobs
- analytics refresh
- materialized view refresh

Never execute these inside HTTP requests.

---

# Monitoring

Monitor:

- query time
- index usage
- cache hit rate after cache is introduced
- lock contention
- slow transactions

Recommended Tools:

- pg_stat_statements
- Prometheus
- Grafana

---

# Database Anti-Patterns

DO NOT:

- use ORM for everything
- execute large reports in HTTP requests
- ignore indexes
- store files inside PostgreSQL
- create giant unbounded tables
- avoid monitoring
- over-cache volatile data
- use huge JOIN-heavy dashboard queries live

---

# Recommended Production Database Architecture

```txt
Frontend
   ↓
Express/NestJS
   ↓
Service Layer
   ↓
Prisma ORM + Raw SQL
   ↓
PostgreSQL Primary DB
   ↓
Partitions
```

Additional Infrastructure:

```txt
Future Redis Cache
BullMQ Queues
Background Workers
Materialized Views
Read Models
Archive Database
```

---

# Engineering Principle

The database is not only for persistence.

It is:

- a transactional engine
- an analytics engine
- a reporting engine
- a financial consistency engine

Modern architecture distributes workloads to the most efficient layer.

---

## Backup & Safety (Neon + R2)

النسخ الاحتياطي جزء حاسم من استراتيجية الصمود. بما أن Neon بيئة مُدارة، لا تعتمد فقط على النسخ التلقائي — خزّن نسخة يومية خارجية في `Cloudflare R2`:

- **استراتيجية بسيطة فعّالة:** جدول مهام (Cron) على سيرفر `DigitalOcean` يقوم بـ `pg_dump` يوميًا ثم يرفع الـ dump إلى `R2` في Bucket مخصّص للنسخ الاحتياطية.

- **مثال Node.js ESM (node-cron + child_process + aws-sdk S3 upload):**

```js
import cron from "node-cron";
import { exec } from "node:child_process";
import fs from "node:fs";
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
  const dumpCmd = `pg_dump "${process.env.BACKUP_DATABASE_URL}" | gzip > /tmp/${filename}`;

  exec(dumpCmd, (err) => {
    if (err) return console.error("dump failed", err);
    // upload to R2
    const body = fs.createReadStream(`/tmp/${filename}`);
    s3.send(
      new PutObjectCommand({
        Bucket: process.env.R2_BACKUP_BUCKET,
        Key: filename,
        Body: body,
      }),
    )
      .then(() => console.log("Backup uploaded to R2"))
      .catch(console.error);
  });
});
```

- **ملاحظات أمنية:** خزّن مفاتيح الـ R2 وبيانات الوصول في secret manager أو envs مقفلة، واحتفظ بسياسة احتفاظ زمنية (retention) مناسبة، وفحص صلاحية النسخ دوريًا. استخدم `DATABASE_URL` كرابط Pooler للتطبيق، واستخدم `BACKUP_DATABASE_URL` كرابط مباشر/قراءة فقط لعمليات `pg_dump`.

## تحسين الأداء (Indexing, Materialized Views, Future Caching)

نقاط سريعة للتطبيق عند إدارة 100+ وحدة/عقار:

- **الفهارس (Indexes):** استعمل `@index` في Prisma أو `CREATE INDEX` في SQL للحقول المستخدمة في WHERE وJOIN وORDER BY (مثل `property_id`, `check_in_date`, `created_at`).

Prisma example:

```prisma
model Booking {
  id         BigInt   @id @default(autoincrement())
  propertyId BigInt   @index
  checkIn    DateTime @index
  createdAt  DateTime @default(now())
}
```

- **Materialized Views:** اجعل العمليات المكلفة تُحسَب مسبقًا ثم كَوّنها كـ materialized views مع تحديث دوري عبر workers.

- **Future Cache Layer (Redis):** لا تضف كاش للقراءات في المرحلة الأولى. ابدأ بالفهرسة وتحسين الاستعلامات وMaterialized Views. لاحقًا، إذا أظهرت المراقبة بطءًا متكررًا في استعلامات التوافر أو لوحات الإدارة أو إحصائيات المدينة، أضف Redis مع TTL مناسب وآلية إلغاء كاش بعد نجاح معاملات قاعدة البيانات.

- **Prisma vs Raw SQL:** استخدم Prisma للحالات اليومية، لكن اعتمد `prisma.$queryRaw` أو وظائف DB المنفذة من خلال workers للاستعلامات التحليلية الثقيلة.

- **مراقبة الاتصالات:** راقب connection count في Neon؛ استخدم pooler في بيئة الإنتاج وقلّل وقت المعاملات لتفادي استهلاك الاتصالات.

---
