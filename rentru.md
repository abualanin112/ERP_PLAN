# تدفّق البيانات

يقوم المبدأ الأساسي لمعمارية الـ ERP على قاعدة: **إدخال واحد واستخدامات متعددة**. تُدخَل البيانات مرة واحدة فقط، ثم تتدفق تلقائياً إلى جميع الوحدات التي تحتاج إليها.
فعلى سبيل المثال، يجب أن يؤدي إنشاء أمر بيع تلقائياً إلى حجز المخزون، وإصدار قائمة التجهيز، وإنشاء الفاتورة، وتسجيل الإيراد محاسبياً، دون الحاجة إلى إعادة إدخال البيانات.

ويعني فهم تدفّق البيانات في تصميم أنظمة ERP الإجابة عن الأسئلة التالية:

- من أين تنشأ البيانات؟
- ما الوحدات التي ستستهلك هذه البيانات؟
- كيف ستتحول البيانات أثناء انتقالها؟
- ما مستوى التأخير المقبول في وصولها؟

وتحدد إجابات هذه الأسئلة متطلبات معمارية التكامل وأداء النظام.

---

# "الفلو الرئيسي" خصيصاً لنظام **Thoth ERP**:

### 1. مرحلة الإدراج والتعاقد الرئيسي (Onboarding & Master Leasing)

تبدأ الرحلة بتأجير العقار من المالك وتجهيزه للبيع للجمهور.

- **التعاقد الرئيسي (Master Lease Agreement):** إدخال العقد السنوي أو الموسمي بين شركتك والمالك في النظام. المالك هنا يُعامل كـ "مُورّد" (Vendor) له التزام مالي ثابت (Accounts Payable)، ولا علاقة له بنسب الإشغال أو إيرادات العملاء.
    
- **إنشاء العقار وجرد الأصول (Property Creation & Inventory):** تعريف المبنى والوحدات (Units)، وتسجيل العفش والأجهزة كأصول ثابتة (Fixed Assets) مملوكة لشركتك أو بعهدتكم.
    
- **إدارة الإيرادات والتسعير الديناميكي (Revenue & Dynamic Pricing):**
    
    - تحديد السعر الأساسي (Base Rate) للمستأجرين.
        
    - يقوم محرك التسعير (Pricing Engine) بتغيير السعر يومياً بناءً على المواسم (Seasonality) والعرض والطلب، لتعظيم هامش الربح فوق الإيجار الثابت المدفوع للمالك.
        

### 2. رحلة محرك الحجز والتسجيل المبكر (Booking Engine & Pre-Registration Flow)

هذا المسار مصمم ليكون سريعاً ومؤتمتاً بالكامل قبل وصول العميل:

- **الاستفسار وعرض السعر (Inquiry & Quote Generation):** يقوم النظام بحساب السعر الإجمالي للعميل.
    
- **التسجيل وسحب البيانات (KYC & OCR Data Extraction):**  يُطلب من العميل إرسال صورة البطاقة فوراً. يقوم النظام بـاستخراج البيانات تلقائياً (OCR) وإنشاء ملف العميل (Customer Profile) في الـ CRM.
    
- **تجميد التقويم (Calendar Block):** قفل مؤقت للوحدة (Temporary Hold) لمنع الحجز المزدوج.
    
- **بوابة الدفع (Payment Gateway):** يتصل السيرفر بـ Paymob، ويُرسل رابط دفع العربون للعميل عبر الواتساب.
    
- **التأكيد والتوليد الآلي للعقد (Confirmation & Auto-Contracting):**
    
    - بمجرد الدفع (عبر الـ Webhook)، تتحول حالة الحجز إلى (Confirmed Booking).
        
    - يقوم النظام **بتوليد عقد الإيجار رقمياً (Digital Lease Generation)** ليكون جاهزاً للطباعة.
        

### 3. عمليات الوصول السريع والمغادرة (Express Check-In & Check-Out)

لأن كل شيء تم إعداده مسبقاً، يتحول الاحتكاك في المكتب إلى ثوانٍ معدودة:

- **الوصول السريع (Express Check-In):** يصل العميل للمكتب. يقوم الموظف فقط بطباعة العقد الجاهز للتوقيع، ويُحصّل المبلغ المتبقي (رقمياً أو نقداً). يضغط زر التأكيد، فتتحول الوحدة إلى حالة (Occupied).
    
- **تسجيل المغادرة (Check-Out):** يقوم العميل بتسليم الشقة. تُراجع حالة العفش، وتتحول حالة الشقة إلى (Vacant - Dirty).
    

### 4. العمليات الميدانية وتجهيز الوحدة (Field Operations & Turnover)

هذا المسار يعمل في الخلفية لضمان دورة تشغيلية لا تتوقف:

- **توليد مهام التجهيز (Turnover Task Generation):** بمجرد خروج العميل، يُصدر النظام تذكرة عمل (Work Order) لعامل النظافة.
    
- **تذاكر الصيانة (Maintenance Ticketing):** فتح طلبات صيانة لأي أعطال وتحميل تكلفتها كمصروف مباشر (Direct Expense) على الوحدة.
    
- **العودة للسوق (Return to Market):** تتحول حالة الشقة إلى (Ready to Rent) فور انتهاء التنظيف.
    

### 5. محاسبة الربحية والالتزامات الثابتة (Master Lease Accounting & Profitability)

بما أن الإيراد بالكامل ملك للشركة، تختلف المحاسبة تماماً عن أنظمة إدارة الأملاك التقليدية:

- **الاعتراف الكامل بالإيراد (100% Revenue Recognition):** كل المبالغ المحصلة من العميل تُسجل فوراً في دفتر اليومية كـ "إيرادات إيجار" (Rental Revenue) مملوكة للشركة.
    
- **الإطفاء اليومي لتكلفة المالك (Daily Amortization):** لمعرفة الربح الحقيقي، يقوم النظام بتقسيم الإيجار السنوي/الموسمي المدفوع للمالك على عدد أيام السنة/الموسم لمعرفة "التكلفة الثابتة لليلة" (Fixed OPEX per night).
    
- **تحليل صافي الربح (Net Profit Analytics):** النظام يطرح (التكلفة الثابتة لليلة + مصروف النظافة/الكهرباء) من (سعر تأجير الليلة للعميل)، ليُظهر للمدير هامش الربح الصافي (Net Margin) لكل وحدة بشكل لحظي.
    
- **جدولة مستحقات الملاك (Accounts Payable - AP):** المالك له جدول دفعات ثابت لا يتأثر بإشغال الشقة، يتم إدارته في موديول المدفوعات.
    

### 💡 الاعتبار المعماري للخبراء: معمارية الخدمات (Service-Oriented Architecture - SOA)

سيعتمد **Thoth ERP** على معمارية تعتمد على فصل المنطق التجاري إلى "خدمات متخصصة" (Services)، حيث تتواصل هذه الخدمات مع بعضها البعض من خلال **(Service Composition)** بدلاً من حشر كل الكود في الـ Controller.

**كيف تعمل هندسة الـ Services في هذا التدفق؟** عندما يقوم العميل بدفع العربون وتأكيد الحجز، يقوم الـ `BookingController` باستدعاء خدمة واحدة فقط، وهي بدورها تدير الأوركسترا بالكامل كالتالي:

1. **`BookingService.confirmBooking()`**: تقوم بتغيير حالة الحجز في قاعدة البيانات إلى "مؤكد".
    
2. ثم تنادي **`ContractService.generatePreContract(customerId, unitId)`**: لتقوم هذه الخدمة بسحب بيانات البطاقة المخزنة (OCR) وتوليد وثيقة العقد كملف PDF رقمي.
    
3. ثم تنادي **`FinanceService.recordIncome(amount)`**: لتقوم هذه الخدمة بفتح قيد محاسبي في دفتر اليومية بالدفعة المُستلمة.
    
4. ثم تنادي **`NotificationService.sendWhatsApp()`**: لإرسال رسالة الترحيب ورقم الشقة للعميل.
    
5. ثم تنادي **`OperationsService.scheduleCleaning(checkOutDate)`**: لجدولة تذكرة النظافة في تقويم العمال بناءً على تاريخ المغادرة.



---
# تدفّق البيانات الأول: Order-to-Cash (دورة المبيعات والتحصيل)

### نقطة البداية (Starting Point)

إدخال طلب إيجار العميل من خلال مكتب المبيعات/الاستقبال (Sales/Office Staff) أو عبر التكامل الفوري مع منصة حجز العقارات عبر الإنترنت (Online Rental/Booking Platform).

### مسار التدفّق (Flow Path)

بيانات الطلب (Request Data) ← وحدة إدارة علاقات العملاء (CRM Module) ← وحدة إدارة العقارات والوحدات والتقويم (Property & Unit Calendar) ← وحدة إدارة العقود والإيجارات (Contract & Lease Management) ← وحدة التشغيل والخدمات والنظافة (Operations & Cleaning Module) ← وحدة الحسابات المدينة والمستحقات (Accounts Receivable - AR) ← وحدة إدارة حسابات الملاك (Multi-Owner Accounting) ← وحدة دفتر اليومية العام (General Ledger - GL).

### مثال عملي (Practical Example)

يقوم أحد العملاء بطلب حجز "الشقة رقم 101" في "مبنى أ" لمدة 5 ليالٍ في ذروة الموسم الصيفي عبر المتجر الإلكتروني أو عبر الاتصال بمكتب فرعي.

يتعامل نظام الـ ERP العقاري الذكي مع العملية على النحو التالي:

### 1. التسجيل والتحقق من الهوية (KYC, Order Intake & CRM)

عند تواصل العميل، يتم إنشاء ملفه (Customer Profile) فوراً. يُطلب من العميل إرسال صورة بطاقة الرقم القومي (عبر الواتساب أو رفعها على رابط الحجز). يقوم النظام بـاستخراج البيانات تلقائياً (OCR) وملء حقول (الاسم، الرقم القومي، العنوان). تُحفظ هذه البيانات المشفرة لمرة واحدة وتُربط بملفه لتسهيل حجوزاته المستقبلية.

### 2. التحقق من التوافر والتسعير الديناميكي (Availability & Dynamic Pricing Check)

تتحقق وحدة إدارة الوحدات (Units Module) من حالة الشقة في التقويم الملوّن (Master Calendar). يقوم محرك التسعير باحتساب سعر الليلة بناءً على العرض والطلب للموسم (مثلاً: 1000 جنيه لليلة × 5 ليالٍ = 5000 جنيه إجمالي).

### 3. الحجز المبدئي وتجميد التقويم (Pending Booking & Calendar Lock)

يُنشئ الموظف "حجزاً مبدئياً". يقوم النظام بتجميد هذه التواريخ للشقة مؤقتاً لمدة 15 دقيقة (Calendar Hold) لمنع أي مكتب آخر من تأجيرها، ريثما يتم تحصيل العربون.

### 4. توليد رابط الدفع الإلكتروني (Automated Payment Link)

يتصل السيرفر ببوابة الدفع (Paymob) لتوليد رابط مشفر بقيمة العربون (مثلاً 1000 جنيه). يُرسل الرابط تلقائياً للعميل عبر أتمتة الواتساب (WhatsApp Automation). او يتم الدفع والعميل متواجد او كاش.

### 5. تأكيد الحجز وإصدار العقد المبدئي (Booking Confirmation & Pre-Contracting)

بمجرد دفع العميل عبر فودافون كاش أو إنستا باي، يلتقط السيرفر الإشارة (Webhook). يتحول لون الحجز في التقويم إلى "مؤكد" (Confirmed). ولأن بيانات الرقم القومي موجودة مسبقاً، يقوم النظام **بتوليد عقد الإيجار رقمياً** (Digital Lease Generation) ويكون جاهزاً للطباعة فوراً. او الارسال عبر الواتس الى العميل.

### 6. الاستلام والتحصيل النهائي (Check-In & Final Collection)

يصل العميل إلى المكتب. العملية هنا تأخذ ثوانٍ معدودة؛ حيث يتم طباعة العقد الجاهز لتوقيعه (لتلبية المسار الورقي)، ويتم تحصيل المبلغ المتبقي (4000 جنيه). يضغط الموظف على زر (Check-In)، فتتحول حالة الوحدة إلى "مشغولة".

### 7. أمر التشغيل والنظافة (Operations & Cleaning Order)

في خلفية النظام، يتم إدراج تذكرة عمل (Work Ticket) مجدولة سلفاً على تقويم عامل النظافة بيوم وتاريخ خروج العميل (Check-Out) لضمان سرعة التجهيز للمستأجر التالي.

### 8. اليومية العامة وتسجيل الإيرادات (General Ledger & Revenue Recognition)

يتم تسجيل الـ 4000 جنيه في [رصيد درج الموظف اللحظي] لتسليمها بنهاية الشيفت. في دفتر اليومية العام (GL)، يتم تسجيل مبلغ الـ 5000 جنيه بالكامل كـ "إيرادات إيجارات" (Rental Revenue) مملوكة لشركتك 100%، وتُخصم منها فقط المصاريف التشغيلية المباشرة للوحدة (مثل رسوم النظافة أو الكهرباء).

## الملخص:

- استخراج **بيانات العميل (Customer Data)** عبر الـ OCR وتخزينها في قاعدة بيانات الـ CRM.
    
- إرسال **بيانات الطلب (التواريخ والوحدة)** للاستعلام من جدول التقويم (Calendar)، ودمجها مع **بيانات محرك التسعير** لتوليد (إجمالي السعر).
    
- إنشاء **سجل حجز (Booking Record)** بحالة "مبدئي"، وتحديث **حقل حالة الوحدة** إلى "مُجمدة".
    
- إرسال **بيانات الدفع (Payment Payload)** لـ API بوابات الدفع (Paymob)، واستقبال (رابط دفع) لتمريره لخدمة WhatsApp.
    
- استقبال **إشعار الدفع (Webhook Data)** لتحديث حالة الحجز إلى "مؤكد"، وتوليد **بيانات العقد الرقمي (Contract Data)** وحفظها في جدول العقود.
    
- تحديث **حقل حالة الوحدة (Unit Status)** في قاعدة البيانات إلى "مشغولة" (Occupied) فور تسجيل الدخول.
    
- استنساخ بيانات الحجز لإنشاء **سجل الفاتورة (Invoice Record)** في جدول الفواتير (Invoices).
    
- ترحيل **بيانات المبلغ المتبقي (Remaining Balance)** كقيد في جدول الحسابات المدينة (Accounts Receivable).
    
- ترحيل **بيانات الإيراد الكلي** و**بيانات التكلفة (COGS)** (إطفاء المالك + النظافة) كقيود مزدوجة في جداول الدفتر العام (General Ledger).
    
- تمرير **بيانات تاريخ المغادرة** من العقد لتوليد **سجل تذكرة عمل (Work Ticket)** في جدول عمليات النظافة والصيانة.
    
- إدخال **بيانات المصاريف المتأخرة (Delayed Expenses)** كقيود جديدة وربطها بـ (Booking ID) لتحديث حقل الربح التراكمي للشقة برمجياً.

---
# تدفّق البيانات الثاني: Procure-to-Pay (دورة الايجار من المالك والسداد)

### نقطة البداية (Starting Point)

إطلاق **بيانات تنبيه اقتراب انتهاء العقد الرئيسي للمالك (Master Lease Expiry Trigger)** نتيجة الوصول إلى تاريخ التجديد، أو إنشاء **طلب تجهيز وصيانة كبير يدوي (Manual Maintenance/Turnover Requisition)** للوحدة.

### مسار التدفّق (Flow Path)

بيانات طلب الاستئجار/التجهيز (Requisition Data) ← نظام الموافقات (Approval Workflow) ← مسودة العقد الرئيسي/أمر العمل (Master Lease/Work Order) ← موافقة المالك/المورد (Vendor/Owner Confirmation) ← محضر استلام الوحدة/الأصول (Unit/Goods Receipt) ← مطالبة السداد الصادرة (Invoice Receipt) ← المطابقة الثلاثية الرقمية (Three-Way Matching Engine) ← جدولة المدفوعات (Accounts Payable) ← الدفتر العام (General Ledger).

### مثال عملي (Practical Example)

يقترب عقد "الشقة رقم 101" مع مالكها الأصلي من الانتهاء، أو تحتاج الشقة إلى عملية إعادة دهان وتجهيز عفش كبرى (Turnover) قبل بداية الصيف. يتعامل السيرفر مع تدفق البيانات كالتالي:

- إطلاق **بيانات مؤشر انتهاء العقد** تلقائياً من جدول العقود الرئيسية (`Master_Leases`) عند بقاء 30 يوماً على نهاية الإيجار الثابت.
    
- توليد **بيانات طلب التجديد أو التجهيز (Procurement Payload)** متضمنة القيمة الإيجارية المستهدفة للمالك أو الميزانية التقديرية للصيانة.
    
- تمرير البيانات عبر موديول الموافقات الرقمية لاعتماد الميزانية الثابتة من مدير النظام (Admin Approval).
    
- تحويل الطلب المعتمد إلى **مسودة عقد رئيسي (Master Lease Draft)** أو أمر شغل فني (Work Order) وإرساله للمالك أو مقاول الصيانة الخارجي.
    
- استقبال وثيقة **موافقة وتوقيع المالك/المورد (Vendor/Owner Sign)** وتحويل حالة العقد في قاعدة البيانات إلى "نشط ومبرم".
    
- تسجيل **بيانات محضر فحص واستلام الشقة/العفش (Check-In Inspection / Asset Receipt)** في جدول الأصول الثابتة بعد المعاينة الميدانية والتأكد من الجودة.
    
- استلام **بيانات مطالبة المالك بالدفع (Landlord Payout Invoice)** أو فاتورة المقاول عن أعمال الدهانات والتجهيز وتخزينها في موديول الفواتير الواردة.
    
- تنفيذ **المطابقة الثلاثية الرقمية (Three-Way Matching Data Engine)** برمجياً للتأكد من تطابق: (شروط العقد المبرم = محضر الفحص والاستلام الميداني = قيمة المطالبة المالية الواردة).
    
- بعد نجاح المطابقة، يتم ترحيل وجدولة **بيانات الدفعات والالتزامات الآجلة (Net 30 Payout)** في جدول الحسابات الدائنة (Accounts Payable) للمالك أو المقاول وفق جدول زمني ثابت.
    
- تحديث **الدفتر العام (General Ledger)** بإثبات الالتزام المالي في (Accounts Payable) وتوزيع تكلفة الصيانة أو القسط الثابت للمالك برمجياً كـ (Fixed OPEX) ليتم إطفاؤها ديناميكياً على أيام تشغيل الوحدة.
    

### الاعتبار المعماري للخبراء (Architectural Consideration)

تعتبر أتمتة **المطابقة الثلاثية الرقمية (Three-Way Matching Engine)** في البروب تك أداة الرقابة الداخلية الأقوى؛ لمنع سداد أقساط ثابتة لملاك عقارات لم تُستلم أصولها بعد، أو صرف مستحقات لمقاولي صيانة قبل تسجيل "محضر فحص الجودة الميداني" للشقة من قِبل المشرف، مما يحمي أرباح الشركة التراكمية من الهدر التشغيلي.


----
# تدفّق البيانات الثالث: Record-to-Report (الإغلاق المالي وإعداد التقارير)

#### نقطة البداية

المعاملات التشغيلية القادمة من جميع وحدات النظام:

- عقود وحجوزات الإيجار.
- التحصيلات والمدفوعات.
- المصروفات.
- التحويلات النقدية بين الفروع والمحاسب والمدير والبنك.
- الإلغاءات والاستردادات.
- التسويات والتعديلات.

#### مسار التدفّق

Rental Contracts وPayments وExpenses وRefunds وCash Transfers وAdjustments → Sub-Ledgers → General Ledger → Financial Statements

#### مثال عملي

أثناء الإغلاق المالي لنهاية الشهر:

- تقوم جميع الوحدات بترحيل القيود إلى General Ledger بشكل تلقائي طوال الشهر.
- يتم تجميع أرصدة العملاء (Accounts Receivable) إن وجدت.
- يتم تجميع أرصدة الموردين (Accounts Payable) إن وجدت.
- يتم تجميع أرصدة الخزن النقدية (الفروع، المحاسب، المدير).
- يتم تجميع أرصدة الحسابات البنكية.
- يتم تجميع العهد النقدية للمدير والموظفين.
- يتم احتساب صافي إيرادات الإيجار بعد خصم الإلغاءات والاستردادات.
- يتم احتساب المصروفات التشغيلية والإدارية.
- يتم دمج جميع البيانات داخل General Ledger.
- يتم استخراج Trial Balance والتأكد من تساوي المدين والدائن تلقائياً.
- يتم إنشاء Snapshot لنتائج الفترة المالية المعتمدة.
- يتم قفل الفترة المحاسبية (Period Lock) ومنع التعديل المباشر على معاملات الفترة.

إعداد:

- Balance Sheet
- Profit & Loss Statement
- Cash Flow Statement
- Revenue by Branch Report
- Occupancy & Season Performance Reports

#### الاعتبار المعماري

في غياب ERP تضطر الشركة إلى تجميع بيانات العقود والتحصيلات والمصروفات والتحويلات النقدية يدوياً من الفروع المختلفة، كما يصعب تتبع أماكن وجود النقدية بين الفروع والمحاسب والمدير والبنك.

أما في ERP فإن جميع العمليات تُسجل فور حدوثها داخل الدفاتر الفرعية وGeneral Ledger في الوقت نفسه، مما يسمح باستخراج القوائم المالية والتقار

---

# تدفّق البيانات الرابع: Plan-to-Rent (التخطيط والتشغيل والتجهيز) ميزة مستقبلية

## نقطة البداية

Sales Forecast أو Reservation Forecast أو Booking Request.


## مسار التدفّق

Demand Forecast → Availability Planning → Unit Allocation → Maintenance & Housekeeping Orders → Unit Preparation → Quality Inspection → Unit Release for Booking → Revenue & Cost Posting

## مثال عملي

توقع النظام إشغال 90% خلال موسم الصيف.

يقوم النظام بما يلي:

- تحليل الحجوزات الحالية والمتوقعة.
- احتساب الوحدات المطلوبة المتاحة خلال الفترة.
- التحقق من الوحدات المحجوزة وغير المتاحة.
- إنشاء أوامر صيانة للوحدات المحتاجة إلى إصلاح.
- إنشاء مهام نظافة وتجهيز للوحدات قبل الاستلام.
- متابعة تنفيذ أعمال الصيانة والنظافة.
- تسجيل نتائج فحص الوحدة قبل تسليمها للعميل.
- تحويل الوحدة إلى حالة Available for Rent.
- استقبال الحجوزات الجديدة بناءً على الطاقة الاستيعابية المتاحة.
- احتساب تكاليف التشغيل والصيانة المرتبطة بالوحدات.
- تحديث General Ledger بإثبات المصروفات والإيرادات الناتجة عن التشغيل.

## الاعتبار المعماري

يتطلب هذا التدفق تكاملاً بين:

- Reservation Management
- Unit Management
- Maintenance Management
- Housekeeping Management
- Financial Management

ومن دون هذا التكامل ستظهر فجوة بين الوحدات المتاحة فعلياً والوحدات القابلة للحجز داخل النظام، مما يؤدي إلى الحجوزات المتعارضة أو انخفاض جودة الخدمة

---

## ميزة مستقبلية: Forecasting إشغال الموسم (Occupancy Forecasting)

### الفكرة العامة

ميزة داخل نظام الـ ERP تقوم بتوقع إشغال موسم المصايف/التأجير قبل حدوثه، اعتماداً على:

- بيانات المواسم السابقة
- الحجوزات الحالية
- سلوك الطلب
- الإلغاءات
- تأثير الأسعار والزمن

> الهدف: مساعدة الإدارة في التخطيط، التسعير، وتوزيع الوحدات قبل الموسم وأثناءه.

# 1. مصادر البيانات (Inputs)

## 1) بيانات تاريخية (Historical Data)

- إشغال كل يوم في المواسم السابقة
- عدد الوحدات المحجوزة
- متوسط الأسعار
- ذروة الموسم وأوقات الركود

## 2) الحجوزات الحالية (On-the-book)

- حجوزات مؤكدة
- حجوزات pending
- طلبات جديدة

## 3) معدلات التحويل (Conversion Rate)

- نسبة تحويل الطلبات إلى حجوزات فعلية

## 4) معدل الإلغاء (Cancellation Rate)

- نسبة الإلغاء حسب الوقت ونوع العميل

## 5) عامل الزمن (Lead Time)

- كلما اقترب التاريخ يزيد الطلب والإشغال

## 6) تأثير السعر (Price Effect)

- رفع السعر يقلل الطلب
- خفض السعر يزيد الطلب

# 2. منطق التوقع (Logic)

يتم حساب الإشغال المتوقع عبر دمج:

```
Base Seasonal Pattern+ Current Bookings+ Expected Conversions- Expected Cancellations± Price & Time Adjustments
```

# 3. الناتج (Outputs)

## توقع الإشغال اليومي/الأسبوعي/الموسمي:

- Expected Occupancy %
- Peak Days
- Low Demand Days
- Fully Booked Periods

## مثال مبسط:

```
100 وحدة60 حجز مؤكد+ 30 طلب جديد × 70% تحويل = 21- 10 إلغاءات متوقعة= 71 وحدة مشغولة→ 71% إشغال متوقع
```

# 4. عرض النظام (UI/UX)

## شاشة Forecast

- خط زمني للموسم (Timeline)
- Heatmap للإشغال:
    - أحمر = ممتلئ
    - برتقالي = عالي الطلب
    - أخضر = متاح
- مؤشرات:
    - Expected Occupancy
    - Booking Pace
    - Cancellation Risk
# 5. الاستخدام العملي

الميزة تستخدم في:

- تحديد الأسعار الديناميكية
- إدارة الضغط على الوحدات
- تجهيز الصيانة قبل الذروة
- منع overbooking
- التخطيط للتوسعات أو رفع الأسعار

# 6. التطور المستقبلي (Future Upgrade)

يمكن تطويرها لاحقاً إلى:

- Dynamic Pricing Engine
- AI Demand Prediction
- Revenue Optimization
- Smart Allocation of Units
# الخلاصة

> Forecasting هو نظام يتنبأ بإشغال الموسم عبر دمج التاريخ + الحجوزات الحالية + سلوك العملاء + تأثير الوقت والسعر، بهدف تحسين التخطيط التشغيلي وزيادة الإيرادات وتقليل الفراغات التشغيلية.

---

### ملخص تدفّقات البيانات في ERP (مُعدّل على بيزنس تأجير العقارات والمصايف)

| العملية               | مصدر البيانات                                                | الوجهة                                                          | متطلبات التأخير | نمط التكامل                      |
| --------------------- | ------------------------------------------------------------ | --------------------------------------------------------------- | --------------- | -------------------------------- |
| Reserve-to-Cash       | طلب حجز / عقد إيجار                                          | Availability, Unit Management, Cash Ledger, AR, GL              | فوري            | API متزامن (Synchronous)         |
| Cash Movement Flow    | تحصيلات، تحويلات، عهد                                        | Cash Locations (فروع، محاسب، مدير، بنك) + GL                    | فوري            | API + Event Driven               |
| Record-to-Report      | جميع المعاملات التشغيلية (حجوزات، مصروفات، تحويلات، إلغاءات) | Sub-ledgers → General Ledger → Financial Statements             | فوري            | Shared Database / Event Sourcing |
| Plan-to-Rent (تشغيلي) | الحجوزات الحالية + التوقعات + الطلب                          | Availability Planning, Unit Status, Maintenance, Cleaning Tasks | يومي (Batch)    | Message Queue / Background Jobs  |
### ملخص تدفّقات البيانات في ERP (مُعدّل على بيزنس تأجير العقارات والمصايف)

|العملية|مصدر البيانات|الوجهة|متطلبات التأخير|نمط التكامل|
|---|---|---|---|---|
|Reserve-to-Cash|طلب حجز / عقد إيجار|Availability, Unit Management, Cash Ledger, AR, GL|فوري|API متزامن (Synchronous)|
|Cash Movement Flow|تحصيلات، تحويلات، عهد|Cash Locations (فروع، محاسب، مدير، بنك) + GL|فوري|API + Event Driven|
|Record-to-Report|جميع المعاملات التشغيلية (حجوزات، مصروفات، تحويلات، إلغاءات)|Sub-ledgers → General Ledger → Financial Statements|فوري|Shared Database / Event Sourcing|
|Plan-to-Rent (تشغيلي)|الحجوزات الحالية + التوقعات + الطلب|Availability Planning, Unit Status, Maintenance, Cleaning Tasks|يومي (Batch)|Message Queue / Background Jobs|### ملخص تدفّقات البيانات في ERP (مُعدّل على بيزنس تأجير العقارات والمصايف)

|العملية|مصدر البيانات|الوجهة|متطلبات التأخير|نمط التكامل|
|---|---|---|---|---|
|Reserve-to-Cash|طلب حجز / عقد إيجار|Availability, Unit Management, Cash Ledger, AR, GL|فوري|API متزامن (Synchronous)|
|Cash Movement Flow|تحصيلات، تحويلات، عهد|Cash Locations (فروع، محاسب، مدير، بنك) + GL|فوري|API + Event Driven|
|Record-to-Report|جميع المعاملات التشغيلية (حجوزات، مصروفات، تحويلات، إلغاءات)|Sub-ledgers → General Ledger → Financial Statements|فوري|Shared Database / Event Sourcing|
|Plan-to-Rent (تشغيلي)|الحجوزات الحالية + التوقعات + الطلب|Availability Planning, Unit Status, Maintenance, Cleaning Tasks|يومي (Batch)|Message Queue / Background Jobs|
### 1. Reserve-to-Cash (بديل Order-to-Cash)

يشمل:

- إنشاء الحجز
- تأكيد العقد
- الدفع أو التحصيل
- إصدار الإيصالات
- الاعتراف بالإيراد

### 2. Cash Movement Flow (إضافة أساسية عندك)

لأنه في بيزنسك النقدية تتحرك بين:

- الفروع
- المحاسب
- المدير
- البنك

وهذا لا يوجد في ERP التقليدي بهذا الوضوح.

### 3. Record-to-Report (محاسبي بحت)

يجمع:

- العقود
- التحصيلات
- المصروفات
- الإلغاءات
- التحويلات النقدية

ثم ينتج:

- GL
- Trial Balance
- Financial Statements
- Season / Branch Reports

### 4. Plan-to-Rent (تشغيلي وليس محاسبي)

يشمل:

- توقع الإشغال
- توزيع الوحدات
- تجهيز الصيانة والتنظيف
- حالة الوحدة (Ready / Occupied / Maintenance)

----
# Backend Architecture Guidelines (ERP - Real Estate / Rentals System)

تحدد Process Flow تجربة المستخدم، بينما يحدد Data Flow اتساق البيانات داخل المؤسسة. وعندما يتم تصميم العمليات دون الاهتمام بتدفّق البيانات، غالباً ما تظهر نسخ متعددة ومتضاربة من نفس المعلومات داخل النظام.

في Monolithic Architecture يتم تبادل البيانات عبر استدعاءات مباشرة أو من خلال قاعدة بيانات مشتركة، مما يجعل الأداء أسرع ويضمن الاتساق عبر معاملات ACID، لكنه يخلق ترابطاً قوياً بين المكونات.

أكثر نقاط الفشل شيوعاً تقع عند الحدود الفاصلة بين ERP والأنظمة الخارجية مثل:

- منصات التجارة الإلكترونية.
- الأنظمة البنكية.

غالباً ما يتم تصميم التدفقات الداخلية بعناية، بينما يُنظر إلى التكاملات الخارجية كمرحلة ثانوية، فتظهر مشكلات مثل:

- فشل مزامنة الطلبات.
- عدم مطابقة المدفوعات.
- اختلاف أرصدة المخزون.

والحل هو التعامل مع التكاملات الخارجية باعتبارها جزءاً أساسياً من تصميم تدفّق البيانات، مع توفير آليات واضحة لمعالجة الأخطاء والمراقبة وإعادة المحاولة، واختبارها تحت الأحمال الفعلية قبل الإطلاق.


## 1. Data Flow Mapping (قبل أي كود)

قبل بناء أي Module لازم يتم تحديد:

- Reservation Flow (حجز → دفع → تأكيد)
- Cash Flow (فرع → محاسب → مدير → بنك)
- Accounting Flow (Transactions → GL → Reports)
- Operational Flow (Unit → Maintenance → Cleaning → Availability)

> كل Flow يتم توثيقه كـ “Domain Journey Map”

## 2. Modular Monolith Structure (DDD)

كل Module مستقل منطقيًا وليس Deployment:

```
/modules  /reservation  /cash-management  /accounting  /units  /maintenance  /reports
```

كل Module يحتوي:

- domain/
- application/
- infrastructure/
- api/

## 3. Service-Based Architecture (بدون Event Driven Core)

بدل Event Sourcing:

> الاعتماد الأساسي = Services + Transactions + Direct Calls

مثال:

```
ReservationService → CashService → AccountingService
```

بدون تعقيد event bus في core logic.

## 4. Real-Time API Layer (Client Facing)

كل العمليات الأساسية تكون:

### Synchronous API

- إنشاء حجز
- دفع
- تعديل عقد
- تحويل نقدية

```
POST /reservationsPOST /paymentsPOST /cash-transfer
```

> العميل يحتاج نتيجة فورية

## 5. Background Processing (Queues فقط للأعمال الثقيلة)

Message Queue يستخدم فقط لـ:

- Daily Close
- Monthly Close
- Season Close
- Forecast Calculation
- Reports generation
- Reconciliation jobs

مثال:

```
queue: accounting.monthly-closequeue: forecast.calculate
```

## 6. Idempotency (مهم جداً)

كل العمليات المالية لازم تكون:

> Safe to retry without duplication

مثال:

```
PaymentRequestId = UUIDIf already processed → return existing result
```

## 7. Data Consistency Strategy

بدون Distributed Transactions:

نستخدم:

- Database Transactions (Prisma)
- Outbox pattern (اختياري خفيف)
- Soft consistency model

مثال:

```
Create Reservation→ Create Cash Record→ Create Journal Entry(inside same DB transaction)
```

## 8. Observability Layer (مهم جداً في ERP)

### لازم تراقب:

#### 1. Data Flow Health

- هل الحجز وصل للـ GL؟
- هل الدفع تم تسجيله؟

#### 2. Failed Integrations

- Payment failed
- Transfer not confirmed

#### 3. Stuck Queues

- Close job لم يكتمل
- Report generation متوقف

## 9. Validation Layer (Before Closing / Posting)

أي عملية مالية تمر على:

```
ValidationService
```

يشمل:

- Balance checks
- Cash mismatch
- Pending approvals
- Missing references

## 10. Accounting Engine (Core Concept)

بدلاً من Event Sourcing:

> كل شيء يتحول إلى Journal Entries

```
Reservation → Revenue EntryPayment → Cash EntryRefund → Contra EntryExpense → Expense Entry
```

## 11. Failure Handling Strategy

أي فشل:

- Retry automatically (idempotent)
- Log failure
- Alert system

## 12. Monitoring Rules (Critical ERP Layer)

### يجب مراقبة:

- Cash mismatch
- Unposted transactions
- Queue backlog
- Failed journal postings
- Period not closed
- Season inconsistencies

# Final Architecture Summary

## System Style

- Modular Monolith
- DDD Boundaries
- Service-Based Core
- Transactional consistency
- Event + Queue only for background jobs

## Flow Model

```
API (Sync Services)   ↓Domain Services   ↓Database Transaction (Prisma)   ↓Journal Entry Engine   ↓GL + Reporting   ↓Queues (Async Jobs)
```

## Key Philosophy

> النظام ليس Event-Driven ERP كامل  
> بل **Service-Oriented Transactional ERP** مع Event-driven layer فقط للـ background processing.

## أهم نقطة في مشروعك

ERP الحقيقي عندك يعتمد على 3 أشياء:

1. **Transaction correctness (الأهم)**
2. **Cash tracking accuracy**
3. **Period closing integrity

---

