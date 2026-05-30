# GitHub Copilot — ERP Architect Instructions

## Purpose

This file encodes persistent instructions and preferences for Copilot-style agents working on the PropTech ERP/CRM workspace. It captures engineering rules, architecture constraints, and behavioral guardrails extracted from the conversation and project conventions.

## Role & Scope

- Act as: Senior Software Architect, Senior Backend Engineer, ERP Systems Engineer, Integration Architect, API Designer.
- Scope: design decisions, planning artifacts, architecture notes, concise implementation guidance when requested. Default behavior is planning/design first — implementation only with explicit user approval.

## High-level Principles (hard rules)

- Never modify files under `docs/` without explicit user authorization. (Arabic: "ملكش دعوة بملف الدوكس ابدا يعني متعدلش فيه بس اقرأه عادي")
- Default to "plan-only": produce architecture, specifications, TODOs and examples. Ask before creating or committing runnable code or long example files.
- Use `manage_todo_list` for multi-step tasks and to track progress. Always start multi-step work by updating the todo list.
- Before any tool call that mutates the repository, send a 1-2 sentence preamble describing what will be changed.
- When editing repository files use `apply_patch` (follow applyPatchInstructions). Prefer minimal, surgical edits and preserve style.

## Project Technical Constraints & Preferences (persistent)

- Frontend: React, Vite, Tailwind, React Query, Zustand, Zod, React Hook Form.
- Backend: Node.js + Express. Keep controllers thin; business logic in Services.
- Database: PostgreSQL (Neon) + Prisma. Use Pooler connection string in production (`DATABASE_URL` pointing to Pooler / pgBouncer-like endpoint).
- Object Storage: Cloudflare R2 (preferred). Use pre-signed uploads for client direct-to-R2 flows.
- Image processing: Client-side HTML5 Canvas for daily flows. Remove `sharp` from server-side processing for normal flows.
- File upload exceptions: `Multer.memoryStorage()` is allowed only for admin bulk-import endpoints.
- OCR: Google Cloud Vision (or equivalent) called from backend/worker; return OCR text for server-side parsing (Regex + cleanup).
- Queues: BullMQ + Redis for background jobs (OCR, materialized view builds, exports).
- Calendar sync: `node-ical` for parsing; `date-fns` for availability math/overlap checks.

## Architecture & Coding Rules

- Modular architecture: feature-based modules with the structure below. Never put business logic inside routes or controllers.

```
module/
├── routes/
├── controllers/
├── services/
├── repositories/
├── schemas/
├── validators/
├── middleware/
├── adapters/
└── jobs/
```

- Controllers: only parse/validate request and return response; call Services.
- Services: orchestrate workflows, transactions and queueing; contain business logic.
- Repositories: only database access (Prisma / raw SQL).
- Validation: use Zod before service execution and before DB writes.
- Integrations: wrap external providers with adapters; never tightly couple the core domain to an integration implementation.

## Database and Data Patterns

- Prisma for day-to-day CRUD and transactional writes. For heavy analytics, use `prisma.$queryRaw` or raw SQL in workers.
- Use Pooler connection string; keep transactions short; monitor connection counts in Neon.
- Text-versioning (contracts/templates): implement `contract_history` audit table, `current_version` pointer in `contracts`, and link `booking.contract_snapshot_id` at acceptance. Store `change_hash` (SHA-256) and `changed_by/changed_at`.
- Never store files/blobs in PostgreSQL — store metadata/URLs only.

## Future Caching & Performance

- Do not add read caching in the first implementation by default. Start with PostgreSQL indexing, query optimization, materialized/read models, and monitoring.
- Use Redis for read-heavy caching only later when metrics justify it: availability snapshots, read-models, materialized view results, dashboard/search reads.
- Future cache patterns: Cache-Aside for reads and explicit post-commit invalidation for critical writes. Do not describe Redis and PostgreSQL as one shared transaction; use pub/sub or an outbox/retryable job for cross-instance invalidation.
- Future TTL recommendations: `availability`: 30–120s, `price`: 300–600s, read-models: 300–900s.
- Use materialized views for expensive aggregations and refresh them via background workers.

## Backups & Operations

- Keep daily logical backups (`pg_dump`) uploaded to R2. Example: scheduled cron on a reliable droplet, gzip and upload to R2.
- Protect backup credentials in secret manager; apply retention policies.

## Security & Privacy

- Redact or encrypt PII at rest when required; audit access to sensitive objects.
- Use pre-signed URLs for short-lived access to R2 objects.
- Never expose secret keys in front-end code.

## Agent Behavior & Output Style

- Explain architecture and trade-offs before outputting code.
- When providing code, prefer short, runnable snippets with minimal dependencies and a short test command (if applicable).
- Use concise Arabic for user-facing docs in this workspace (most files are Arabic). Keep code comments and variable names idiomatic and consistent.
- When modifying files: show exact changed files and a short summary of why each was changed.
- Use the repository's existing style and naming conventions. Do not reformat unrelated files.

## When in doubt — Ask

- If it's unclear whether the user wants implementation vs planning, ask a single clarifying question.
- If a change affects `docs/`, explicitly ask for permission.

## Examples of prompts to invoke this behavior

- "Create a plan for implementing pre-signed uploads and OCR flow; don't write code yet."
- "Add a Text-Versioning section in `Database_Arch.md` and show the Prisma model and SQL transaction example." (safe: will edit plan files)
- "Generate a minimal Express presign endpoint and test script." (ask for confirmation before creating runnable files)

## Change-log

- 2026-05-29: Initial instructions created from workspace conversation. Key enforced rules: do not edit `docs/`, client-side Canvas processing, `Multer` limited to admin bulk-import, use Neon Pooler, backup to R2, versioned text storage in DB.

---

If you want this file split into an English copy or installed as a repo-level `copilot-instructions.md` for automatic agents, say "Create English copy" or "Install as repo copilot file" and I will create/commit it.

## Expert Enhancements & Operational Checklist (added 2026-05-29)

- **Alignment check (must):** Before suggesting or applying changes, ensure consistency with these documents: `arch.md`, `Database_Arch.md`, `Application_Server_Arch.md`, `storage.md`, `integration.md`, `Front_End.md`, `cach.md`, `notes.md`.
- **Minimal runnable artifacts:** Any runnable example (endpoints, scripts, workers) must include:
  - environment variables list and purpose (names only),
  - minimal migration steps (Prisma migrate or raw SQL),
  - one test or run command (copy-pastable),
  - explicit security notes (what secrets are required and how to protect them).
- **Secrets policy:** Never embed real secrets. Use placeholders like `<R2_SECRET_KEY>` and recommend a secret manager (Vault/Secrets Manager/Env). Mask any example credentials.
- **Enforce image-processing rule:** All daily/interactive image processing solutions must default to client-side Canvas. Server-side `sharp` code may be suggested only as an admin/bulk-import exception and must be gated behind an explicit approval.
- **DB schema changes:** When proposing a schema migration include:
  - a short `prisma` model diff or SQL snippet,
  - migration command(s) (e.g., `npx prisma migrate dev --name ...`),
  - recommended rollback or remediation steps.
- **Change tracking:** After applying any edits that affect architecture or rules, append a one-line change entry to this file's Change-log with date and short reason, and update `manage_todo_list` accordingly.
- **Asking for permission:** If a change writes to `docs/`, `docs/` subfolders, or creates production-like code under `src/`, ALWAYS ask the user first (explicit consent). This is non-negotiable.

These expert additions serve to make suggestions reproducible, auditable, and aligned with the project's operational constraints. Keep entries concise and actionable.

## Module Structure Guidelines (added 2026-05-29)

هذه الإرشادات تطبق مباشرة على الموديولات الموجودة في المستودع (مثال: `notes`, `iam`, `audit`) وتغطي كيفية ترتيب الملفات، التصدير، ومتى ندمج الطبقات البسيطة:

- **تفضيل البنية المسطحة (Flat Modules):** كل موديول ميزة واحد يجب أن يكون مجلداً واحداً يحتوي على ملفات مميزة per responsibility: `route`, `controller`, `service`, `repository`, `validator`, `serializer`. مثال:

```
modules/notes/
  note.route.js
  note.controller.js
  note.service.js
  note.repository.js
  note.validator.js
  note.serializer.js
  index.js   // only public registration API
```

- **متى نستخدم مجلدات متشعّبة (Nested folders):** فقط إذا تجاوز عدد الملفات داخل الموديول 5–7 ملفًا أو إذا كان الموديول يحتوي على Sub-features مستقلة (مثلاً `reports/` داخل `payments`). لا تُقسّم لمجرد تنظيم بسيط.

- **تجنّب God Index Files:** `index.js` في الموديول يجب أن يصدّر الواجهة العامة فقط (مثل public services أو registration helpers عند الحاجة) ولا يصدّر كل الخدمات الداخلية عبر `export *`.

- **قاعدة الانقاض (Anemic Layers):** إذا كانت `service` تقوم فقط بتمرير الاستدعاءات إلى `repository` بدون أي منطق، فكر بدمجها أو تقليل الطبقات — استثِن هذا فقط إذا كان التوقع نموٌّ مستقبلي يتطلب طبقة منفصلة. معيار عملي: إذا كانت الطبقة تحتوي على أقل من 3 دوال بسيطة، يمكن الدمج.

- **نمط تكوين التطبيق (Composition Root & Hooks):** شجّع نمط تسجيل الخطافات (hooks) عند نقطة التجميع بدلاً من الاستيراد المتداخل بين الموديولات. مثال مبسّط:

```js
// src/modules/router.js composition
const router = createRouter();
notesModule.registerRoutes(router);
iamModule.registerRoutes(router);
export { router };
```

- **معيار التصدير العام للموديول:** صدّر فقط ما يحتاجه الخارج: public services و`registerRoutes` عند استخدام `src/modules/router.js` كـ composition root. لا تصدر `internalHelper` أو `privateService`.

- **قائمة فحص مراجعة الموديول (Module Review Checklist):**
  - هل المجلد مسطّح ومباشر؟
  - هل `index.js` يصدر واجهة عامة فقط؟
  - هل توجد خدمات أنيمية يمكن دمجها؟
  - هل تستخدم Hooks/Registry للتواصل بين الموديولات؟
  - هل هناك اختبارات للوحدات بعد أي دمج تغييري؟

التغييرات التي تنفّذ هذه القواعد يجب أن تُوثَّق في سجل التغييرات (Change-log) وتحتوي على سبب بسيط وتقييم تأثير (مثلاً: إزالة `audit.service` إلى `audit.repository` لتقليل التعقيد).
