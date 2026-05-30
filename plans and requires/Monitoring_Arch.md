# Monitoring & Proactive Alerting Plan

## Purpose

This plan defines a practical monitoring strategy for the PropTech ERP backend without over-engineering early. The goal is to catch production issues before users report them, while keeping the maintenance routine lightweight.

Start with:

- Built-in alerts from Neon, Cloudflare, and DigitalOcean.
- Sentry for backend code errors.
- Pino structured logs from the application.
- A simple weekly/monthly maintenance routine.

Delay until later:

- Grafana and Prometheus dashboards.
- Centralized infrastructure telemetry.
- Advanced SIEM-style security monitoring.

## Monitoring Principles

- Alerts should be proactive, not manual dashboard checking.
- Production `error` logs should map to actionable alerts.
- Logs must be structured and safe: no tokens, passwords, cookies, authorization headers, raw request bodies, financial payloads, or full Prisma entities.
- Use request context from AsyncLocalStorage where available: `requestId`, `userId`, and trace/correlation IDs.
- Monitor each component with the tool closest to it before introducing a central dashboard.

## 1. Proactive Alerting Strategy

Do not manually open Neon, DigitalOcean, and Cloudflare dashboards every hour. Configure alerts once and let the platforms notify you when resource usage approaches risky thresholds.

### Neon Alerts

Configure alerts from the Neon dashboard.

Recommended thresholds:

- CPU usage >= 80%.
- Storage usage >= 80%.
- Connection usage >= 80% of the safe pool limit.
- Slow query or query latency alerts if available in the selected Neon plan.

Operational response:

- Check recent deployments and traffic changes.
- Inspect slow queries with `EXPLAIN ANALYZE`.
- Add or adjust indexes based on actual query patterns.
- Review connection pooling and long transactions.

### Cloudflare R2 Alerts

Use Cloudflare Alerts from the main Cloudflare dashboard.

Recommended thresholds:

- Storage usage above an expected monthly budget.
- Bandwidth above an expected monthly budget.
- Sudden request spikes on private buckets.
- Unusual 4xx/5xx rates if exposed through Cloudflare analytics.

Operational response:

- Check whether uploads/downloads are expected.
- Confirm that private buckets are not accidentally public.
- Review signed URL expiration settings.
- Investigate large files or abnormal client retry behavior.

### DigitalOcean Droplet Alerts

Use DigitalOcean Monitoring on the Droplet running the backend, workers, backup jobs, or Nginx.

Recommended thresholds:

- RAM usage >= 90%.
- CPU usage >= 85% for sustained periods.
- Disk usage >= 80%.
- Disk I/O saturation if available.
- Restart or downtime alerts.

Operational response:

- Check process memory growth.
- Inspect worker queues and long-running jobs.
- Review Nginx/API logs for spikes.
- Scale vertically only after checking code-level causes.

## 2. Sentry: Code Error Monitoring

Sentry is the most important early monitoring tool for the Express + Prisma backend.

Sentry is not primarily for CPU or RAM. It is for application errors:

- Unhandled exceptions.
- Failed Prisma calls.
- Neon connection failures.
- Worker errors.
- API route crashes.
- Unexpected integration failures.

Why it matters:

- It reports the error type.
- It shows the file and stack trace.
- It groups repeated issues.
- It captures runtime context around the failure.
- It saves debugging time by showing the real production failure path.

Recommended setup:

- Install Sentry in the Express API entrypoint.
- Capture uncaught exceptions and unhandled promise rejections.
- Attach safe context only: `requestId`, `userId`, route name, job name, and environment.
- Do not send raw request bodies, tokens, cookies, authorization headers, or PII.
- Enable release tracking when deployments become regular.

Alert rules:

- Immediate alert for new `error` groups in production.
- Immediate alert for repeated database connection errors.
- Daily or weekly digest for low-volume non-critical errors.
- Separate alerts for worker crashes if workers run as independent processes.

Free plan:

- The free Sentry plan is enough for the early ERP stage and should be used before building any custom monitoring dashboard.

## 3. Application Logs

The backend should use structured Pino logs.

Log level policy:

- `error`: system failures, uncaught exceptions, transaction failures, infrastructure failures. Should trigger alerts.
- `warn`: degraded but recoverable states, retries, suspicious activity, failed logins.
- `info`: important business and lifecycle events.
- `debug`: development troubleshooting only; disabled in production by default.
- `trace`: deep diagnostics only; disabled in production by default.

Safe logging rules:

- Never log `req.body` as a whole.
- Never log passwords, refresh tokens, JWTs, cookies, authorization headers, financial payloads, or raw Prisma entities.
- Always pass full error objects to the logger, e.g. `logger.error({ err, event: "db.connection.failed" }, "Database connection failed")`.
- Prefer event names such as `booking.created`, `auth.login.failed`, `worker.job.failed`, and `cache.invalidation.failed`.

## 4. Component Metrics To Watch

### Database

- Query latency.
- Slow queries.
- Connection count.
- Storage growth.
- Lock contention.
- Index hit rate.

### API Server

- API response time.
- Error rate.
- Request throughput.
- Memory usage.
- CPU usage.
- Process restarts.

### Future Redis / Cache

- Track these only after Redis cache is introduced:
  - Cache hit rate.
  - Redis latency.
  - Memory usage.
  - Evicted keys.
  - Cache invalidation failures.

### Queues / Workers

- Queue depth.
- Job latency.
- Failed jobs.
- Retry counts.
- Dead-letter queue size.
- Worker process restarts.

### Object Storage

- Storage usage.
- Bandwidth usage.
- Upload/download request count.
- Signed URL failure rate.
- Unexpected public access or permission errors.

### Integrations

- Webhook failures.
- Provider timeout rate.
- Retry counts.
- Circuit breaker open events.
- Idempotency conflict rate.

## 5. Future Central Dashboard

Use a consolidated dashboard only when moving between multiple dashboards becomes a real operational cost.

Future stack:

- Prometheus for metrics collection.
- Grafana for dashboards.
- Node exporter for Droplet metrics.
- PostgreSQL exporter for database metrics when appropriate.
- Redis exporter for cache metrics later, only after Redis cache becomes operationally important.

Warning:

- Do not set up Grafana now. It is a project by itself.
- Start with provider alerts + Sentry + structured logs.
- Add Grafana later when there are multiple services, multiple workers, or regular operational reviews.

## 6. System Maintenance Routine

### Simple Engineering Routine

لكي لا يضيع وقتك في الإدارة، اتبع هذا الروتين البسيط:

| التردد | المهمة |
| --- | --- |
| يوميًا | لا تفعل شيئًا يدويًا. اعتمد على التنبيهات. |
| أسبوعيًا | ادخل على Sentry لمدة 10 دقائق لترى إذا كانت هناك أخطاء متكررة، حتى لو لم تسقط النظام. |
| شهريًا | راجع استهلاك الموارد وBill/Usage Report في Neon وCloudflare وDigitalOcean للتأكد من أنك لا تزال ضمن المنطقة الآمنة. |

### Detailed Maintenance Routine

| Frequency | Task | Time Budget |
| --- | --- | --- |
| Daily | Do nothing manually. Depend on alerts. | 0 minutes |
| Weekly | Open Sentry and review repeated errors, even if they did not bring the system down. | 10 minutes |
| Weekly | Check failed queue jobs and recurring worker warnings. | 10 minutes |
| Monthly | Review Neon, Cloudflare, and DigitalOcean usage/billing reports. | 20 minutes |
| Monthly | Check database growth and storage growth. Add cache memory trends only after Redis cache is introduced. | 20 minutes |
| After every deployment | Watch Sentry for new errors and verify API health checks. | 10 minutes |

## 7. Alert Severity Matrix

| Severity | Examples | Response |
| --- | --- | --- |
| Critical | API down, database unreachable, repeated payment failure, worker crash loop | Immediate action |
| High | CPU/storage above 90%, repeated Prisma errors, dead-letter queue growth | Same day |
| Medium | Slow query appears, R2 usage spike, or cache hit rate drops after cache is introduced | Review in weekly routine |
| Low | One-off recoverable warning, expected validation noise | Track only |

## 8. Implementation Roadmap

### Phase 1: Now

- Configure Neon CPU/storage/connection alerts.
- Configure Cloudflare R2 storage/bandwidth alerts.
- Configure DigitalOcean RAM/CPU/disk alerts.
- Add Sentry to Express and workers.
- Confirm Pino logs are structured and redacted.

### Phase 2: After First Production Usage

- Add explicit event names to critical logs.
- Add Sentry release tracking.
- Add queue failure alerts.
- Review slow database queries weekly.
- Add cache hit-rate reporting if Redis is actively used.

### Phase 3: Later

- Add Prometheus and Grafana only if the built-in dashboards become inconvenient.
- Add infrastructure exporters.
- Build a single operational dashboard for API, DB, Redis, workers, and storage.

## 9. Acceptance Checklist

- Neon alerts are enabled at 80% CPU/storage and safe connection thresholds.
- Cloudflare R2 alerts are enabled for storage and bandwidth.
- DigitalOcean alerts are enabled for RAM, CPU, disk, and downtime.
- Sentry is connected to the Express API.
- Sentry is connected to workers if workers run separately.
- Logs use Pino, include safe context, and avoid sensitive payloads.
- Weekly and monthly routines are documented and followed.
- Grafana is explicitly deferred until operational complexity justifies it.
