نظرًا لأن النظام يعتمد على نموذج **الاستئجار لإعادة التأجير (Master Lease / Rental Arbitrage)** مع وجود دورة حسابية صارمة تشمل المصاريف المؤجلة وصلاحيات المستخدمين، فإن تقسيم الجداول الاحترافي يقع في مجموعات مترابطة:

### 1. مجموعة العقارات والوحدات (Property & Inventory Management)

في جدران قواعد البيانات العلائقية، يتم فصل المبنى عن شققه تماماً عبر علاقة رأس بأطراف (One-to-Many).

- **جدول المباني (Properties):**
    
    - `id` (Primary Key)
        
    - `name` (VARCHAR) - اسم المبنى أو العشة
        
    - `location` (VARCHAR) - المنطقة في رأس البر
        
    - `gps_location` (VARCHAR)
        
    - `created_at` (TIMESTAMP)
        
- **جدول الوحدات (Units):**
    
    - `id` (Primary Key)
        
    - `property_id` (Foreign Key -> Properties) - ربط بال مبنى
        
    - `unit_number` (VARCHAR) - رقم الشقة
        
    - `type` (VARCHAR) - (غرفتين، صالة، أرضي، علوي)
        
    - `status` (ENUM) - حالة الوحدة الحالية (متاحة، مشغولة، صيانة، تنتظر تنظيف)
        
    - `base_price` (DECIMAL) - السعر الأساسي لليلة
        
    - `created_at` (TIMESTAMP)
        

### 2. مجموعة الملاك والعقود الرئيسية (Owners & Master Leases)

هنا يتم إدارة الطرف الأول من البيزنس (المصاريف الثابتة)، حيث يُعامل المالك كمورد مستقل عن حركات المستأجرين.

- **جدول الملاك (Owners):**
    
    - `id` (Primary Key)
        
    - `name` (VARCHAR)
        
    - `phone` (VARCHAR)
        
    - `created_at` (TIMESTAMP)
        
- **جدول العقود الرئيسية مع الملاك (Master_Leases):**
    
    - `id` (Primary Key)
        
    - `unit_id` (Foreign Key -> Units)
        
    - `owner_id` (Foreign Key -> Owners)
        
    - `total_amount` (DECIMAL) - إجمالي مبلغ الإيجار المدفوع للمالك (سنوي/موسمي)
        
    - `start_date` (DATE)
        
    - `end_date` (DATE)
        
    - `daily_amortized_cost` (DECIMAL) - التكلفة اليومية المطفأة للشقة (يحتسبها النظام تلقائياً)
        

### 3. مجموعة العملاء والحجوزات (CRM & Booking Engine)

- **جدول العملاء (Customers):**
    
    - `id` (Primary Key)
        
    - `name` (VARCHAR)
        
    - `phone` (VARCHAR) - فريد لعدم تكرار العميل
        
    - `national_id` (VARCHAR) - الرقم القومي (المستخرج عبر الـ OCR عند التسجيل)
        
    - `address` (VARCHAR)
        
    - `customer_tag` (ENUM) - (VIP، عادي، مشاكل)
        
    - `created_at` (TIMESTAMP)
        
- **جدول الحجوزات (Bookings):**
    
    - `id` (Primary Key)
        
    - `unit_id` (Foreign Key -> Units)
        
    - `customer_id` (Foreign Key -> Customers)
        
    - `user_id` (Foreign Key -> Users) - الموظف الذي قام بالحجز
        
    - `start_date` (DATE)
        
    - `end_date` (DATE)
        
    - `total_amount` (DECIMAL) - السعر الإجمالي المحتسب بناءً على التسعير الديناميكي
        
    - `advance_payment` (DECIMAL) - العربون المدفوع
        
    - `status` (ENUM) - (مبدئي، مؤكد، مدفوع بالكامل، ملغي)
        
    - `created_at` (TIMESTAMP)
        
- **جدول عقود الإيجار الرقمية (Lease_Contracts):**
    
    - `id` (Primary Key)
        
    - `booking_id` (Foreign Key -> Bookings)
        
    - `digital_serial` (VARCHAR) - السيريال التلقائي للسيستم
        
    - `paper_serial` (VARCHAR) - السيريال الورقي الذي يدخله الموظف يدوياً للمطابقة
        
    - `status` (ENUM) - (نشط، منتهي)
        
    - `created_at` (TIMESTAMP)
        

### 4. النواة المالية ودليل الحسابات (ERP Core & General Ledger)

لتطبيق القيود المزدوجة ومحاسبة الاستحقاق بدقة، وفصل الإيرادات عن المصاريف المتأخرة (مثل عمولات السماسرة وأجور العمال اللاحقة)، نستخدم جدول الحسابات التراكمي مع جدول المعاملات.

ولاً: الهيكلة البرمجية للجداول المالية (SQL Schema Tables)

#### - جدول دليل الحسابات (Ledger_Accounts)

يمثل شجرة الحسابات الثابتة للنظام (Chart of Accounts). حسابات مثل ذمم العملاء (Accounts Receivable) وتكلفة التشغيل (COGS) هي سجلات أساسية (Rows) داخل هذا الجدول وليست جداول منفصلة.

- `id` (Primary Key)
    
- `account_code` (VARCHAR) - كود الحساب المحاسبي الثابت (مثال: `1101` للخزينة، `1201` للذمم/AR، `4101` للإيرادات، `5101` للـ COGS).
    
- `name` (VARCHAR) - اسم الحساب (خزينة وبوابات الدفع، ذمم المستأجرين، إيرادات التأجير، تكلفة تشغيل الوحدات).
    
- `type` (ENUM) - نوع الحساب لتحديد طبيعته المحاسبية (`Asset`, `Liability`, `Equity`, `Revenue`, `Expense`).
    
- `current_balance` (DECIMAL) - الرصيد التراكمي اللحظي الحالي للحساب (Running Balance) والذي يتحدث تلقائياً مع كل حركة لتوفير سرعة فائقة في القراءة.
    
#### - جدول الفواتير (Invoices)

المستند التجاري والقانوني الذي يثبت المعاملة مع المستأجر ويتحكم في شروط السداد والآجال التشغيلية.

- `id` (Primary Key)
    
- `booking_id` (Foreign Key -> Bookings) - ربط مباشر بملف حجز المستأجر.
    
- `invoice_number` (VARCHAR) - رقم فاتورة فريد ومسلسل آلياً للتدقيق المالي.
    
- `status` (ENUM) - حالة الفاتورة التشغيلية (`Draft`, `Posted`, `Paid`, `Partially_Paid`, `Cancelled`).
    
- `issue_date` (DATE) - تاريخ إصدار الفاتورة للعميل.
    
- `due_date` (DATE) - تاريخ الاستحقاق النهائي (وهنا يتم تفعيل شروط السداد مثل **Net 30** للإيجارات طويلة الأجل والشركات، أو الفوري عند الدخول للإيجار اليومي).
    
- `total_amount` (DECIMAL) - إجمالي القيمة الصافية للفاتورة.
    
#### - جدول دفتر القيود اليومية - رأس القيد (Journal_Entries)

يمثل الحدث المالي ككتلة واحدة متكاملة (مثل حدث تسجيل دخول مستأجر، أو حدث دفع عمولة مؤجلة لسمسار، أو تسوية أجور عمال).

- `id` (Primary Key)
    
- `entry_number` (VARCHAR) - رقم القيد المحاسبي الآلي الفريد للمراجعة (Audit Trail).
    
- `invoice_id` (Foreign Key -> Invoices, Nullable) - يربط بالفاتورة التشغيلية الأصلية إن وجدت.
    
- `booking_id` (Foreign Key -> Bookings, Nullable) - يربط بالحجز الأساسي لجمع كافة التدفقات المالية والمصاريف المستقبلية اللاحقة تحت مظلة حجز واحد.
    
- `description` (TEXT) - البيان الكلي لسبب القيد المحاسبي.
    
- `date` (TIMESTAMP) - تاريخ ترحيل القيد الفعلي للدفاتر.
    

#### - جدول قيود الأستاذ المساعد - أطراف القيد (Journal_Items)

هنا تسكن النواة الحقيقية لأفكار **الدائن والمدين (Debit / Credit)**. كل سطر يمثل طرفاً في القيد، ويشترط النظام أوتوماتيكياً توازن إجمالي المدين مع الدائن لكل رأس قيد (`journal_entry_id`).

- `id` (Primary Key)
    
- `journal_entry_id` (Foreign Key -> Journal_Entries) - ربط برأس القيد لضمان توازن أطراف المعاملة بالمليم.
    
- `account_id` (Foreign Key -> Ledger_Accounts) - الحساب المحدد المتأثر من شجرة الحسابات (مثل توجيه الحركة لحساب AR أو الخزينة).
    
- `unit_id` (Foreign Key -> Units, Nullable) - ربط مباشر بالشقة/الوحدة المستهدفة، وهو الميزان الخارق الذي يتيح حساب صافي ربح وخسارة كل شقة ديناميكياً شاملة مصاريفها اللاحقة.
    
- `debit` (DECIMAL) - الجانب المدين (+) للأصول والمصروفات، ويكون صفراً إذا كان السطر دائناً.
    
- `credit` (DECIMAL) - الجانب الدائن (+) للالتزامات والإيرادات، ويكون صفراً إذا كان السطر مديناً.
    
- `details` (VARCHAR) - الشرح والتفنيد الخاص بهذا السطر من القيد (مثال: تحصيل عربون، إثبات متبقي آجل).
        

### 5. التشغيل والخدمات والصيانة (Operations & Maintenance)

- **جدول التذاكر والتشغيل (Maintenance_And_Cleaning_Tickets):**
    
    - `id` (Primary Key)
        
    - `unit_id` (Foreign Key -> Units)
        
    - `type` (ENUM) - (تنظيف دوري، صيانة طارئة، تجهيز موسم)
        
    - `status` (ENUM) - (مفتوحة، قيد التنفيذ، منتهية)
        
    - `description` (TEXT)
        
    - `assigned_to` (Foreign Key -> Users, Nullable) - العامل المسؤول
        
    - `cost` (DECIMAL) - تكلفة الصيانة أو المواد المستخدمة (تُرحل لاحقاً كـ Expense في جدول المعاملات)
        
    - `created_at` (TIMESTAMP)
        

### 6. الصلاحيات والتحكم بالأدوار (RBAC - Role Based Access Control)

لحماية البيانات المالية وفصل عهد الموظفين عن حسابات المحاسب الكبير والآدمن، نعتمد على جداول علاقة (Many-to-Many):

- **جدول المستخدمين (Users):** `id`, `name`, `email`, `password_hash`, `is_active`
    
- **جدول الأدوار (Roles):** `id`, `name` (مدير، محاسب، موظف مكتب، عامل)
    
- **جدول الصلاحيات الدقيقة (Permissions):** `id`, `name` (مثل: `create_booking`, `view_finance_kpis`, `post_delayed_expense`)
    
- **جدول ربط المستخدم بالدور (User_Roles):** `user_id`, `role_id` (يسمح بتكديس الأدوار للمستخدم الواحد)
    
- **جدول ربط الدور بالصلاحية (Role_Permissions):** `role_id`, `permission_id`
    

### ميزة هذا التقسيم العلائقي في الاستعلامات (SQL Joins):

عندما يطلب المدير تقرير الربح والخسارة الكلي لشقة معينة شاملاً المصاريف المتأخرة، لن يحتاج النظام لإعادة حساب الدفاتر، بل سينفذ استعلاماً بسيطاً وسريعاً يجمع فيه كل الإيرادات والمصاريف المرتبطة بـ `unit_id` في جدول `Transactions`:

SQL

```
SELECT 
    SUM(CASE WHEN type = 'Credit' AND (category = 'rent_revenue') THEN amount ELSE 0 END) AS Total_Revenue,
    SUM(CASE WHEN type = 'Debit' THEN amount ELSE 0 END) AS Total_Expenses,
    (SUM(CASE WHEN type = 'Credit' AND (category = 'rent_revenue') THEN amount ELSE 0 END) - 
     SUM(CASE WHEN type = 'Debit' THEN amount ELSE 0 END)) AS Net_Profit
FROM Transactions
WHERE unit_id = ?;
```

هذا الهيكل يضمن أن نظام الحسابات يظل مرناً للغاية؛ فلو أُضيف أي مصروف للسمسار أو فني الصيانة بعد شهر من خروج المستأجر، بمجرد ربطه برقم الشقة والحجز، سيتم احتسابه فوراً وتحديث كشف الحساب التراكمي بدقة متناهية.