# frontend-stack-react-zustand-zod.md

# ERP / CRM Frontend Architecture

## React + JavaScript + Zustand + Zod + Full Stack Open Style

---

# الهدف

بناء Frontend احترافي وقابل للتوسع لنظام ERP / CRM باستخدام:

- React
- JavaScript
- Vite
- TailwindCSS
- Zustand
- Zod
- React Query
- React Hook Form

بنفس فلسفة وأسلوب Full Stack Open مع Architecture مناسبة للأنظمة الكبيرة.

---

# Core Stack

## Framework

- React
- JavaScript
- Vite

---

# Frontend Philosophy

الواجهة مسؤولة عن:

- Rendering
- UX
- Forms
- Dashboards
- API Communication

For calendar UI (viewing appointments and availability) the frontend consumes iCal-derived slots via the API; heavy date computations use `date-fns` on the backend, while the frontend renders slots and interactions.

Image capture / OCR flows:

- For capturing `national ID` or utility bills (water/electricity), provide a mobile-friendly capture UI that uploads the image securely to the backend.
- Use an in-app camera experience with instant preview and retake options; send images to the backend for OCR rather than running OCR in the browser to reduce fragmentation.
- Hybrid flow (Client-side compression + Direct R2 upload):

1. الضغط داخل المتصفح (Frontend Compression):

- بمجرد أن يلتقط الموظف صورة البطاقة أو فاتورة، استخدم مكتبة خفيفة مثل `browser-image-compression` لتقليل الحجم (مثلاً من عدة ميجابايت إلى ~200KB) مع الحفاظ على وضوح الأرقام.

2. طلب الإذن من السيرفر (Pre-signed URL):

- المتصفح يطلب من الـ API رابط رفع مؤقت لـ R2. الـ API يتحقق من صلاحية المستخدم والـ booking (إن وُجد) ثم يصدر الرابط.

3. الرفع المباشر (Direct Upload to R2):

- المتصفح يرفع الملف المضغوط مباشرة إلى Cloudflare R2 باستخدام الرابط المؤقت.

4. المعالجة النهائية (Trigger OCR):

- بعد نجاح الرفع، يُخطر المتصفح الـ API برابط الكائن داخل R2. عندها يقوم الـ API بإرسال رابط الصورة إلى خدمة OCR (Google Vision) أو بدلاً من ذلك يضع مهمة في صف لمعالجتها لاحقًا.

هذا الأسلوب يقلل تحميل السيرفر ويستفيد من قدرة المعالجات على الأجهزة المحمولة لتقليل استهلاك الشبكة.

For image uploads (property photos, IDs, bills) the frontend MUST perform compression and watermarking via `HTML5 Canvas` (client-side) and upload the final asset directly to Cloudflare R2 using pre-signed URLs. The backend does NOT run `sharp` for daily flows.

Exceptions: only one permitted server-side path:

- **Bulk Imports (CSV/XLSX)**: an admin-only endpoint may accept file uploads (use `Multer.memoryStorage()`), parse rows server-side and insert into the DB. All other image processing stays client-side.

القاعدة: المسار الافتراضي لرفع الصور اليومية يبقى client-side compression → direct upload؛ الخادم مسؤول فقط عن التحقق، إصدار الروابط، وتلقي إشعارات ما بعد الرفع.

وليست مسؤولة عن:

- Business Logic
- Pricing Rules
- Financial Calculations
- Inventory Decisions

---

# Frontend Architecture Style

## المطلوب

Feature / Module Based Architecture

---

# Recommended Folder Structure

```txt id="32kq8m"
src/
│
├── app/
│   ├── router/
│   ├── layouts/
│   ├── providers/
│   └── store/
│
├── modules/
│   ├── auth/
│   ├── users/
│   ├── sales/
│   ├── inventory/
│   ├── accounting/
│   ├── crm/
│   └── reports/
│
├── shared/
│   ├── components/
│   ├── ui/
│   ├── hooks/
│   ├── services/
│   ├── utils/
│   ├── lib/
│   └── schemas/
│
├── assets/
└── main.jsx
```

---

# Structure لكل Module

```txt id="c9x3pe"
sales/
│
├── pages/
├── components/
├── hooks/
├── store/
├── services/
├── api/
├── schemas/
└── routes/
```

---

# Styling System

## استخدم

- TailwindCSS

---

# State Management

# Zustand

## يستخدم لـ:

- Sidebar State
- Theme
- Filters
- Dialogs
- UI Preferences
- Wizard Steps
- Temporary UI State

---

# لماذا Zustand؟

- بسيط جدًا
- سريع
- Boilerplate قليل
- ممتاز لمشاريع React الحديثة
- أخف وأسهل من Redux

---

# Example

```jsx id="ofj6s0"
import { create } from "zustand";

const useSidebarStore = create((set) => ({
  isOpen: true,

  toggle: () =>
    set((state) => ({
      isOpen: !state.isOpen,
    })),
}));
```

---

# ممنوع استخدام Zustand لـ:

- API Caching
- Server State
- Pagination Data
- Remote Data Synchronization

---

# Server State Management

## استخدم

- TanStack Query (React Query)

---

# React Query مسؤول عن:

- Fetching
- Caching
- Refetching
- Pagination
- Synchronization

---

# API Layer

## استخدم

- fetch + React Query

---

# API Structure

```txt id="8ec2m9"
shared/services/api/
│
├── axios.js
├── auth-api.js
├── sales-api.js
├── inventory-api.js
└── accounting-api.js
```

---

# Authentication

## النظام

- JWT Authentication
- Refresh Tokens
- Google OAuth

---

# Forms Architecture

## الأدوات

- React Hook Form
- Zod

---

# لماذا Zod؟

لأن ERP يحتوي على:

- بيانات كثيرة
- نماذج معقدة
- Validation متكرر
- Dynamic Forms

---

# Zod مسؤول عن:

- Validation
- Schema Definitions
- Form Safety
- API Validation Helpers

---

# Example

```jsx id="1rk1i5"
import { z } from "zod";

export const salesOrderSchema = z.object({
  customerId: z.string(),
  total: z.number().positive(),
});
```

---

# React Hook Form + Zod

```jsx id="1pwvpg"
const form = useForm({
  resolver: zodResolver(salesOrderSchema),
});
```

---

# قواعد مهمة

## Validation الحقيقي يكون داخل Backend

Validation داخل Frontend:

- لتحسين UX فقط.

---

# ممنوع داخل Frontend

- Credit Limit Decisions
- Inventory Rules
- Financial Calculations
- Permission Decisions الأساسية

---

# Tables System

## استخدم

- TanStack Table

---

# مطلوب دعم

- Pagination
- Sorting
- Filtering
- Export
- Row Selection
- Column Visibility

---

# Dashboard System

## الأدوات

### Tremor

لـ:

- KPI Cards
- Financial Dashboards
- Analytics UI

### Chart.js

لـ:

- Charts
- Reports
- Interactive Graphs

---

# Dashboard Examples

## Financial Dashboard

- Revenue
- Expenses
- Profit
- Cash Flow

---

# Inventory Dashboard

- Low Stock
- Inventory Value
- Fast Moving Products

---

# Sales Dashboard

- Conversion Rate
- Monthly Revenue
- Sales Funnel

---

# Routing System

## استخدم

- React Router

---

# Layout System

## أنواع Layouts

- Auth Layout
- Dashboard Layout
- Admin Layout

---

# Role-Based UI

الواجهة يجب أن تتغير حسب:

- User Role
- Permissions

---

# Example

```jsx id="4snpkg"
{
  can("sales.create") && <CreateOrderButton />;
}
```

---

# مهم جدًا

الواجهة ليست Security Layer.

حتى لو تم إخفاء زر:

- الـ Backend يجب أن يتحقق أيضًا.

---

# Reports Architecture

## Frontend Responsibilities

الواجهة مسؤولة عن:

- Report Filters
- Date Pickers
- Export Buttons
- Dashboard Visualizations
- Preview Tables
- Charts & KPIs

---

## Backend Responsibilities

الخادم مسؤول عن:

- PDF Generation
- Excel Generation
- CSV Exports
- Financial Reports
- Tax Reports
- Invoice PDFs
- Large Dataset Processing

---

# لماذا يتم إنشاء التقارير على السيرفر؟

لأن ذلك يوفر:

- Security
- Better Performance
- Centralized Business Logic
- Accurate Financial Calculations
- Permission Enforcement
- Large File Processing

---

# Recommended Frontend Tools

## Dashboards & Visualization

- Tremor
- Chart.js
- TanStack Table

---

# Recommended Backend Tools

## Excel

- ExcelJS
- xlsx

## PDF

- PDFKit
- Puppeteer
- pdf-lib

## CSV

- csv-writer

---

# Recommended Flow

```txt id="b0r2sl"
Frontend
   ↓
Select Filters
   ↓
Call Report API
   ↓
Backend Generates File
   ↓
Stream File / Signed URL
   ↓
Frontend Downloads File
```

---

# Important Rule

Frontend يجب ألا يحتوي على:

- Financial Calculations
- Tax Logic
- Accounting Rules
- Inventory Valuation Logic

كل ذلك يجب أن يكون داخل الـ Backend.

---

# Performance Requirements

## مطلوب

- Lazy Loading
- Code Splitting
- Suspense
- Memoization
- Virtualized Tables

---

# UX Requirements

## النظام يجب أن يحتوي على:

- Loading States
- Error States
- Empty States
- Toast Notifications
- Confirmation Dialogs

---

# Notifications

## الأنواع

- Success
- Error
- Warning
- Info

---

# Enterprise UI Principles

## المطلوب

- سرعة التنقل
- وضوح البيانات
- أقل عدد Clicks
- Responsive Design
- Keyboard Navigation

---

# مستقبل النظام

يجب أن يكون جاهزًا لاحقًا لـ:

- Multi-Tenant
- Feature Flags
- Dynamic Themes
- WebSockets
- Realtime Dashboards

---

# قواعد معمارية مهمة

## Frontend مسؤول عن:

- Rendering
- UX
- Forms
- Dashboards
- API Communication

---

## Backend مسؤول عن:

- Business Logic
- Inventory Logic
- Financial Rules
- Pricing
- Permissions
- Workflows

---

# Recommended Future Research

- Bulletproof React
- Zustand Patterns
- Zod Schema Design
- React Query Patterns
- Enterprise Dashboard UX
- Feature Based Architecture
- RBAC UI Systems
- Frontend Clean Architecture
