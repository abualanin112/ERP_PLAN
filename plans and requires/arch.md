# Modern PropTech ERP/CRM Architecture Guide

## Express + Prisma + PostgreSQL

> This document explains the architectural philosophy and engineering patterns used in modern scalable PropTech systems.

---

# Core Philosophy

Modern enterprise systems do NOT place all logic inside:

- Express APIs
  OR
- PostgreSQL

Instead:

> Each workload is executed in the most efficient layer.

---

# High-Level Architecture

```txt
Frontend
   ↓
Express/NestJS API Layer
   ↓
Service Layer / Domain Logic
   ↓
Prisma ORM + Raw SQL
   ↓
PostgreSQL
```

Additional Infrastructure:

```txt
BullMQ Queues
Background Workers
Materialized Views
Read Models
Stored Procedures
Partitioning
Future Redis Cache
```

Component-level planning:

- See [business/README.md](business/README.md) for business requirements, business logic, domain workflows, rules, and UX translation preparation.
- See [ERP_Core_Components.md](ERP_Core_Components.md) for the core ERP components, common scaling challenges, monitoring metrics, and practical cache adoption guidance.
- See [Monitoring_Arch.md](Monitoring_Arch.md) for proactive alerts, Sentry, safe logging, maintenance routines, and the future Grafana path.

---

# Layer Responsibilities

# 1. Express / NestJS Layer

Responsible for:

- Authentication
- Authorization
- Validation
- API orchestration
- Workflow coordination
- Integrations
- Event publishing
- Domain/business rules

Examples:

- Create Booking
- Create Payment
- User Management
- Availability Validation
- Notification Flows

Best Practices:

- Keep controllers thin
- Use service layer
- Use DTO validation
- Use dependency injection
- Separate domain logic from infrastructure

---

# 2. Prisma ORM

Use Prisma for:

- CRUD operations
- Relations
- Transactions
- Type-safe queries

Examples:

- Create Property
- Update Booking
- Fetch Users
- Manage Relations

Best Practices:

- Avoid deeply nested Prisma queries
- Avoid using Prisma for analytics/reporting
- Use Prisma transactions carefully
- Keep database schema normalized

---

# 3. Raw SQL

Use Raw SQL for:

- Heavy reports
- Aggregations
- Financial calculations
- Dashboard metrics
- Optimized performance queries

Examples:

- Revenue reports
- Occupancy calculations
- Owner profitability
- Top properties analytics

Example:

```sql
SELECT
  property_id,
  COUNT(*) bookings,
  SUM(total_price) revenue
FROM bookings
GROUP BY property_id
ORDER BY revenue DESC;
```

Best Practices:

- Benchmark queries
- Add proper indexes
- Avoid ORM-generated heavy queries
- Use EXPLAIN ANALYZE frequently

---

# 4. Stored Procedures

Use Stored Procedures ONLY for:

- Financial closing
- Revenue distribution
- Cost rollups
- Bulk operations
- Heavy transactional logic

Examples:

- Monthly owner payouts
- Tax closing
- Commission calculations

Example:

```sql
CALL close_monthly_revenue(period_id);
```

Best Practices:

- Keep procedures focused
- Make procedures idempotent
- Add audit logging
- Avoid moving all business logic into PostgreSQL

---

# 5. Background Workers

Background workers execute long-running tasks outside HTTP requests.

Use Workers for:

- PDF generation
- Financial reports
- Calendar sync
- Email sending
- Data imports
- Analytics refresh
- Bulk processing

Architecture:

```txt
API
 ↓
Queue
 ↓
Worker
 ↓
Database
```

Recommended Stack:

- BullMQ
- Redis

Best Practices:

- Never run heavy jobs inside HTTP requests
- Add retries
- Add dead-letter queues
- Monitor job failures

---

# 6. Queues

Queues manage asynchronous tasks reliably.

Use Cases:

- Emails
- Notifications
- Webhooks
- Sync jobs
- Report generation
- Integrations

Example Flow:

```txt
Create Booking
  ↓
Queue:
 - Send Email
 - Generate Invoice
 - Notify Owner
 - Sync Calendar
```

Best Practices:

- Retry failed jobs
- Use priorities
- Separate queues by domain
- Monitor queue latency

---

# 7. Future Redis Cache Layer

Read caching is not part of the first implementation by default. Add Redis cache later when monitoring shows repeated read pressure or slow endpoints after indexing/query optimization.

Future Redis cache can be used for:

- Frequently accessed data
- Search caching
- Dashboard caching
- Property listing caching

Examples:

- Featured properties
- City statistics
- Popular searches
- Availability snapshots

Best Practices when cache is introduced:

- Cache read-heavy data only
- Avoid caching rapidly changing transactional data
- Set expiration policies
- Invalidate cache correctly

---

# 8. Read Models

Read Models are optimized structures for dashboards and analytics.

Instead of:

- Many JOINs
- Expensive live calculations

Use:

- Precomputed read tables/views

Example:

```txt
owner_dashboard_stats
```

Contains:

- Total revenue
- Occupancy rate
- Monthly bookings
- Upcoming guests

Best Practices:

- Separate write models from read models
- Use read models for dashboards only
- Refresh asynchronously

---

# 9. Materialized Views

Materialized Views store precomputed query results.

Use Cases:

- KPIs
- Financial summaries
- Revenue reports
- Analytics dashboards

Example:

```sql
CREATE MATERIALIZED VIEW city_revenue AS
SELECT
 city,
 SUM(total_price)
FROM bookings
GROUP BY city;
```

Refresh:

```sql
REFRESH MATERIALIZED VIEW city_revenue;
```

Best Practices:

- Refresh periodically
- Use for expensive aggregations
- Combine with Redis caching later only if the materialized view is still a read bottleneck

---

# 10. Partitioning

Partitioning splits massive tables into smaller pieces.

Use For:

- bookings
- payments
- audit_logs
- financial_transactions

Recommended Strategy:

- RANGE partitioning by date

Example:

```sql
PARTITION BY RANGE (created_at)
```

Best Practices:

- Use monthly/yearly partitions
- Add indexes per partition
- Avoid over-partitioning
- Start only when data becomes large

---

# 11. Archiving

Archive old inactive data.

Example:

- Keep last 2 years active
- Move older records to archive storage

Use Cases:

- Historical bookings
- Audit logs
- Old invoices
- Closed financial periods

Best Practices:

- Separate active/archive tables
- Keep archive searchable
- Automate retention policies

Related implementation notes:

- See [archive.md](archive.md) for a focused implementation guide on object storage (Cloudflare R2): versioning, lifecycle rules, object lock (WORM), encryption at rest/in-transit, and pre-signed URL examples (including an Express snippet). Include `archive.md` in operational runbooks for storage configuration and legal retention policies.

---

# 12. Specialized Queries

Specialized queries are optimized for specific use cases.

Example:
Property Search:

```sql
SELECT *
FROM properties
WHERE city = 'Damietta'
AND price_per_night <= 100
AND amenities @> ARRAY['wifi'];
```

Best Practices:

- Optimize per screen/use-case
- Avoid generic mega queries
- Use indexes strategically

---

# PropTech-Specific Examples

# Booking Flow

Handled by:

- Express/NestJS
- Prisma

Includes:

- Availability validation
- Pricing logic
- Discounts
- Payment orchestration

---

# Owner Revenue Closing

Handled by:

- PostgreSQL Stored Procedure
- Background Worker

Includes:

- Revenue calculation
- Commission deduction
- Tax handling
- Financial snapshots

---

# Dashboard Analytics

Handled by:

- Materialized Views
- Read Models
- Future Redis Cache

Includes:

- Occupancy rate
- Revenue trends
- Top properties
- Booking metrics

---

# Search Engine

Handled by:

- Specialized SQL Queries
- Future Redis Cache

Includes:

- City filters
- Amenities filters
- Price ranges
- Availability windows

---

# Scalability Philosophy

Small CRUD Apps:

- ORM-centric
- Request-response only

Enterprise Systems:

- Hybrid architecture
- Distributed workloads
- Background processing
- Specialized data layers

---

# Golden Rule

Use:

- Express/NestJS for workflows and business orchestration
- PostgreSQL for heavy data processing
- Redis for future read caching when metrics justify it
- Queues for async jobs
- Workers for long-running tasks

---

# Anti-Patterns to Avoid

DO NOT:

- Put all business logic inside PostgreSQL
- Execute heavy jobs in HTTP requests
- Use ORM for everything
- Cache highly volatile transactional data
- Build giant generic queries
- Ignore indexing
- Ignore database monitoring

---

# Recommended Stack

Backend:

- Express.js or NestJS

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

Monitoring:

- pg_stat_statements
- Prometheus
- Grafana

Infrastructure:

- Docker
- Nginx
- CI/CD

---

# Final Engineering Principle

Modern enterprise architecture is NOT:

> "Where should logic go?"

Instead:

> "Which layer can execute this workload most efficiently, reliably, and scalably?"
