# Future Caching Architecture Development Plan

**Context**: Read caching is a future performance phase for the PropTech ERP project, not an immediate implementation requirement. The first phase should rely on PostgreSQL indexing, query optimization, materialized/read models where needed, and monitoring. Redis may still be used earlier for BullMQ, rate limiting, token blacklists, or short-lived coordination state if those features are implemented.

## 0. Adoption Rule

Do not implement read caching by default in the first release. Start this plan only when monitoring shows one or more of the following:

- Repeated read endpoints are placing measurable pressure on PostgreSQL.
- API response time is affected by repeated expensive queries.
- Dashboard/search/read-model endpoints become slow despite indexing and query optimization.
- The system grows from hundreds of daily transactions to thousands and selected read caching can reduce database load.

## 1. Future Caching Strategy Analysis

- Review existing Redis usage across the codebase, if Redis is already used for queues, rate limiting, or token blacklists.
- Identify bottlenecks and sources of stale data.

## 2. Keyspace Design

- Adopt a colon‑delimited naming convention: `namespace:entity:id:attribute`.
  - Example keys:
    - `branch:1:unit:15:availability`
    - `price:unit:15:date:2026-08-15`
- Document the convention and train the team.
- Enable bulk invalidation by scanning matching patterns safely, e.g., `SCAN branch:*:unit:15:*` then `UNLINK` matching keys. Do not pass patterns directly to `DEL`.

## 3. Data Structure Selection

- **HASHES** for entities where individual fields are updated (e.g., unit price).
  - Use `HSET` / `HGETALL` to modify a single field without rewriting the whole object.
- **SORTED SETS** for ordered lists such as:
  - Cheapest properties
  - Latest listings
  - Upcoming bookings
- **STRINGS** for simple tokens (e.g., session IDs).
- **LISTS** for temporary queues.

## 4. Invalidation Strategy

- **Future Cache‑Aside** (read-through) for selected read-heavy endpoints:
  1.  Attempt to read from Redis.
  2.  On miss, fetch from PostgreSQL and populate Redis.
- **Future explicit invalidation after critical writes** (e.g., price changes):
  - Commit the PostgreSQL transaction first, then invalidate Redis keys and publish an invalidation event.
  - Do not describe Redis and PostgreSQL as one shared transaction. For high-risk workflows, use an outbox table or retryable job so invalidation is not lost after the DB commit.
- When cache is introduced, encapsulate all cache logic in a `CacheService` class to keep controllers clean and allow future cache-engine swaps.

## 5. Eviction Policy

- Set `maxmemory-policy` to `allkeys-lru`.
- Define appropriate `maxmemory` limits based on server capacity.
- Assign TTLs per key type:
  - Frequently changing data: 5‑10 minutes.
  - Relatively static data: up to 24 hours.

## 6. Anti‑Patterns to Avoid

- **Never** use `KEYS *` in production; replace with `SCAN` for safe iteration.
- Avoid storing large blobs (e.g., full images) in Redis; store only metadata and reference external storage.
- Always set a TTL to prevent memory bloat.

## 7. Testing & Observability

- Before implementing cache, measure baseline database query time and API response time.
- After implementing cache, add load tests to measure cache hit-rate and latency under stress.
- Monitor Redis metrics (latency, hit-rate, memory usage) only after Redis caching is active.

## 8. Documentation & Training

- Keep this plan as the working caching reference unless the project owner explicitly approves adding a dedicated `docs/architecture/caching.md` later.
- Conduct a workshop for the development team on Redis best practices before implementing read caching in production.

---

### Existing Guidelines (Retained)

1.  **Keyspace Design** – Use colon‑delimited keys for easy bulk deletion.
2.  **Data Structure Selection** – Prefer HASHES and SORTED SETS over raw JSON strings.
3.  **Invalidation Strategy** – Cache‑Aside for reads and explicit post-commit invalidation for critical writes.
4.  **Eviction Policy** – `allkeys-lru` with appropriate memory limits.
5.  **Anti‑Patterns** – Avoid `KEYS *`, large objects, and missing TTLs.

By following this structured plan later, the caching layer will become maintainable, performant, and ready for scaling without adding premature complexity now.

## أمثلة تنفيذية مستقبلية وشفرة (لوقت تطبيق الكاش لاحقًا)

### مثال Cache-Aside (Node.js + node-redis)

```js
import { createClient } from "redis";

const client = createClient({ url: process.env.REDIS_URL });
await client.connect();

async function getAvailability(propertyId, date) {
  const key = `availability:property:${propertyId}:date:${date}`;
  const cached = await client.get(key);
  if (cached) return JSON.parse(cached);

  const data = await fetchAvailabilityFromDb(propertyId, date); // تطبيقية
  // TTL قصير لأن بيانات التوفر تتغيّر بسرعة
  await client.set(key, JSON.stringify(data), { EX: 60 });
  return data;
}
```

### إبطال الكاش بعد تغيير متسجل (Invalidate after update)

```js
// نفّذ التغيير داخل المعاملة في DB
await prisma.$transaction(async (tx) => {
  await tx.booking.create({ data: bookingPayload });
});

// بعد نجاح المعاملة: احذف المفتاح ذي الصلة
await client.unlink(`availability:property:${propertyId}:date:${date}`);

// لتنسيق مع مثيلات متعددة، انشر رسالة إبطال بعد نجاح المعاملة
await client.publish(
  "cache-invalidate",
  JSON.stringify({ type: "key", value: `availability:property:${propertyId}:date:${date}` }),
);
```

### اشتراك الإبطال (Subscriber)

```js
const sub = client.duplicate();
await sub.connect();
await sub.subscribe("cache-invalidate", async (message) => {
  const event = JSON.parse(message);

  if (event.type === "key") {
    await client.unlink(event.value);
    return;
  }

  if (event.type === "pattern") {
    for await (const key of client.scanIterator({ MATCH: event.value, COUNT: 100 })) {
      await client.unlink(key);
    }
  }
});
```

### نصائح TTL مقترحة

- `availability:*` = 30–120 ثوانٍ (اعتمادًا على الحمل).
- `price:*` = 300–600 ثانية (5–10 دقائق).
- `read-models` أو نتائج `materialized views` = 300–900 ثانية.

### حذف أنماط آمن (بدون KEYS \*)

استخدم `SCAN` مع `UNLINK` أو Pipeline لحذف مفاتيح مطابقة لنمط بشكل آمن وبأداء مقبول.

### تسخين الكاش (Pre-warming) - لاحقًا

قم بمهام خلفية بعد النشر أو قبل ذروة الاستخدام لتسخين أهم المفاتيح (top properties, top cities) لتجنب موجة من الـ cache misses.

### مراقبة وقياس

- راقب `hit_ratio`, `latency_ms`, `used_memory` و`evicted_keys`.
- ربط Redis مع Prometheus/Grafana أو مستعمل موفّر خدمة لإعداد تنبيهات.

### الأمان والتوفر

- استخدم TLS وAUTH، قيّد الوصول عبر VPC أو قواعد الجدار الناري.
- لا تعتمد على نسخة واحدة — استخدم Redis Cluster/managed provider للـ HA.

هذه الأمثلة تكمل الإرشادات العامة أعلاه وتوفّر نماذج عملية للتطبيق داخل `Express`/`Prisma`.

## استخدامات متقدمة ونمطيات هندسية لـ Redis (خبير)

في هذه الفقرة نغطي استخدامات Redis المهنية. بعضها قد يكون مبكرًا للبنية التحتية مثل BullMQ وRate Limiting وToken Blacklisting، أما read caching/search/dashboard cache فيبقى مرحلة مستقبلية.

### 1. حظر التوكن (Token Blacklisting)

عند استخدام JWT، احتفظ بقائمة سوداء (blacklist) في Redis لتمكين إبطال التوكن فورًا (مثلاً حالة استقالة موظف).

نمط تخزين بسيط: `bl:<tokenId>` مع `EX` يساوي المدة المتبقية للتوكن.

مثال (تخزين وفحص):

```js
// بعد استخراج tokenId من التوكن (jti أو hash)
await client.set(`bl:${tokenId}`, "1", { EX: ttlSeconds });

// middleware
const isBlacklisted = await client.exists(`bl:${tokenId}`);
if (isBlacklisted) return res.status(401).json({ error: "Token revoked" });
```

ملاحظة: خزّن فقط معرف التوكن أو hash وليس التوكن الكامل.

### 2. الحد من معدل الطلبات (Rate Limiting)

استخدم `INCR` مع `EXPIRE` لتنفيذ Rate Limiter سريع وذو تكلفة منخفضة.

مثال بسيط (per-user or per-IP):

```js
const key = `rl:${scope}:${id}:${windowSeconds}`; // scope=ip|user
const current = await client.incr(key);
if (current === 1) await client.expire(key, windowSeconds);
if (current > limit) {
  // Block or throttle
}
```

تعزيزات:

- استخدم LUA script لعمل check+incr+ttl بشكل ذري (atomic) وتحسين الأداء.
- نفّذ استجابة ذكية (429 + Retry-After header).

### 3. الطوابير والمهام الخلفية (BullMQ)

للمهام الطويلة (PDF, bulk emails, OCR orchestration) استخدم BullMQ المبني على Redis.

مثال موجز:

```js
import { Queue, Worker } from "bullmq";

const queue = new Queue("reports", {
  connection: { url: process.env.REDIS_URL },
});

// إضافة وظيفة
await queue.add("generate-report", { bookingId: 123 });

// عامل تنفيذ
new Worker(
  "reports",
  async (job) => {
    // job.data.bookingId
    await generateReport(job.data.bookingId);
  },
  { connection: { url: process.env.REDIS_URL } },
);
```

مزايا BullMQ: retries, delays, rate-limiting per worker، UI عبر Arena أو BullBoard.

### 4. القفل الموزع (Distributed Locking) — حماية الحجز

لا تعتمد على `SETNX` بسيط دون حدود زمنية؛ استخدم نمط القيمة الفريدة مع `PX` ثم تحقّق قبل الإفراج لسلامة التحرير.

مثال آمن (single Redis node):

```js
const lockKey = `lock:unit:${unitId}`;
const lockVal = `${process.pid}-${Date.now()}`;
const acquired = await client.set(lockKey, lockVal, { NX: true, PX: 30000 });
if (!acquired) throw new Error("Locked");

try {
  // execute booking within DB transaction
} finally {
  const cur = await client.get(lockKey);
  if (cur === lockVal) await client.del(lockKey);
}
```

ملاحظة: إذا كان لديك Redis Cluster، استخدم مكتبة `redlock` لتنفيذ بروتوكول Redlock الموزع.

### 5. Pub/Sub للتنبيهات اللحظية

استخدم Pub/Sub لنشرات تحميل منخفضة: عند الانتهاء من Job أو إتمام الحجز انشر رسالة تُشترك بها خوادم الويب أو خادم WebSocket لعرض إشعارات فورية.

مثال اشتراك ونشر:

```js
// publisher
await client.publish(
  "notifications",
  JSON.stringify({ type: "booking.created", bookingId }),
);

// subscriber (in a ws process)
const sub = client.duplicate();
await sub.connect();
await sub.subscribe("notifications", (msg) => {
  const payload = JSON.parse(msg);
  // push to connected clients
});
```

### 6. جدول ملخص: أين تضع ماذا في Redis؟

| الحالة (Use Case)                |         هيكل Redis | السبب                                        |
| -------------------------------- | -----------------: | -------------------------------------------- |
| تفاصيل وحدة عقارية (حقول متعددة) |             `HASH` | تحديث حقول فردية دون إعادة تحميل كامل الكائن |
| قائمة انتظار المهام              |   `List` / Streams | FIFO وميزات استهلاك متسلسلة                  |
| الأسعار حسب التاريخ              |       `Sorted Set` | ترتيب وقيم (score) بحسب السعر/التاريخ        |
| جلسة المستخدم / توكن             |           `String` | Token أو مرجع بسيط سريع القراءة              |
| قائمة التوكنات المحظورة          | `String` مع TTL لكل توكن | إبطال تلقائي عند انتهاء صلاحية التوكن وبحث سريع بـ `EXISTS` |
| قيود الطلب (Rate limit)          | `String` (counter) | INCR + EXPIRE منخفض التكلفة                  |

### 7. نصائح هندسية نهائية

- ابدأ بالاحتياجات الفعلية: Queues وToken Mgmt وRate Limiting عند الحاجة، ثم أضف Read Cache لاحقًا بعد ظهور مؤشرات أداء حقيقية.
- عزل الوصول إلى Redis داخل `CacheService` أو `QueueService` لتسهيل التبديل والاختبار.
- استخدم Scripts/Lua للعمليات متعددة الخطوات التي تحتاج لذرّية.
- قيّم الحاجة لنسخ Redis (Replica/Cluster) مبكراً إذا كنت تعتمد على Pub/Sub وRedlock.
- ضع مراقبة متقدمة: `latency`, `used_memory`, `hit_ratio`, `blocked_clients`, `evicted_keys`.

هذه الإضافات تُكمّل دليل الكاش المستقبلي وتوفر خارطة طريق تنفيذية للتطبيق الآمن عندما تثبت الحاجة له في بيئة الإنتاج.
