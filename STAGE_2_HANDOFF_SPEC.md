# STAGE 2 HANDOFF SPECIFICATION

وثيقة تسليم تشغيلية بين:

- STAGE 2 — FINANCIAL CALCULATION ENGINE
- STAGE 3 — CLIENT DATA MAPPING + MASTER TEMPLATE INJECTION

المصدر: التنفيذ الحالي للمحرك في `financial_engine/` واختباراته ووثائقه الموجودة.
لا تحسين. لا اقتراح. لا منطق مالي جديد.
إذا لم توجد معلومة في التنفيذ: **NOT DEFINED**.

تاريخ الاستخراج من النسخة الحالية: 2026-08-19.

---

## SECTION 1 — ENGINE IDENTITY

**Name (package / CLI):** Financial Calculation Engine  
**Module:** `financial_engine`  
**CLI:** `python -m financial_engine` · `python run_stage2.py`  
**Package version (`financial_engine.__version__`):** `2.0.0`  
**ResultBundle.engine_version default (actual field):** `1.0.0`  
هذه قيمتان موجودتان معاً في التنفيذ. المحرك لا يوحّدهما. لا يُفترض أنهما شيء واحد.

**Master citation constant (`__master__`):**  
`MASTER SPECIFICATION — template is the absolute reference`

**Function:**  
تنفيذ تجميعات قالب Business Sales / Profitability Dashboard المعرفة في MASTER SPECIFICATION فقط (Dataset → Table1 → Pivot Cache logic → Analytical sheets → KPIs → DashBoard GETPIVOTDATA).

**What it does:**

- يحمّل Table1 (CSV / JSON / XLSX / list-of-dicts) بلا اختراع صفوف.
- يجمّد المصدر (`FrozenDataset`) ويحسب `source_hash`.
- ينفّذ 22 معادلة مسجّلة فقط عبر `REGISTRY`.
- يصنّف كل ناتج رسمي: `SOURCE` / `PYTHON_DERIVED` / `NOT_AVAILABLE`.
- يميّز على مستوى الخلية: `ZERO` / `MISSING` / `PRESENT` / `NOT_AVAILABLE`.
- يسجّل Audit + Calculation Log.
- يعيد الحساب في مسار مستقل (Reconciliation).
- يكتب تقارير JSON/MD بعد `run_pipeline`.

**What it does not do:**

- لا يبني Excel ولا يحقن Master Arabic Template.
- لا يعرّب ولا يصمّم Dashboard ولا يغيّر نوع/مصدر الرسوم.
- لا ينفّذ معادلات غير مسجّلة.
- لا يفترض `Profit = Net Sales − COGS` ولا أي هوية صفّية غير موجودة في MASTER.
- لا يحوّل العملة ولا يفترض وحدة قياس.
- لا يستنتج مرشحات من Timeline/Slicers.
- لا يعتبر Fixture بيانات عميل في مسار الإنتاج.

**Limits:** انظر SECTION 10.

---

## SECTION 2 — SOURCE RULE

طبقات التصنيف في التنفيذ طبقتان. لا تُخلطا.

### A. Classification (ناتج رسمي / خلية رقمية مستخدمة)

من `financial_engine/types.py` → `Classification`:

| القيمة | المعنى في التنفيذ |
|---|---|
| `SOURCE` | رقم موجود في Table1 بعد التحليل، وصالح للاستخدام |
| `PYTHON_DERIVED` | رقم محسوب من SOURCE بمعادلة في Formula Registry |
| `NOT_AVAILABLE` | لا قيمة رسمية. `value` يجب أن يكون `None` |

قيد التنفيذ: `NOT_AVAILABLE` مع رقم ≠ ممنوع (`ClassificationErrorGuard`).  
`PYTHON_DERIVED` أو `SOURCE` بلا قيمة ≠ ممنوع.

### B. Presence (إشغال الخلية)

من `Presence` + `infer_presence()` في `dataset.py`:

| القيمة | المعنى في التنفيذ |
|---|---|
| `ZERO` | رقم SOURCE صريح يساوي 0. ملاحظة حقيقية. تدخل SUM |
| `MISSING` | خلية فارغة / رمز فراغ في عمود موجود (`""`, `None`, `n/a`, `#NAME?`, `-`, …) |
| `PRESENT` | رقم غير صفر، أو نص بُعد غير فارغ |
| `NOT_AVAILABLE` | لا يمكن الاستخدام: عمود غائب، نص غير قابل للتحليل دون تخمين، عملة مجهولة، boolean، NaN/Inf |

### الفرق الإلزامي

- **ZERO ≠ MISSING.** الصفر المكتوب في العميل يُجمع. الفراغ لا يُحوَّل إلى 0.
- **MISSING ≠ NOT AVAILABLE.** الفراغ حالة خلية. NOT AVAILABLE حالة ناتج أو خلية غير قابلة للاستخدام.
- **PYTHON_DERIVED** ليس SOURCE. مطابقة GETPIVOTDATA تبقى PYTHON_DERIVED.
- ناتج مجموعة بلا صفوف = `NOT_AVAILABLE` وليس 0.
- عمود غائب = `NOT_AVAILABLE` (`MISSING_COLUMN`) وليس 0.
- `SUM` لمجموعة فيها أصفار مصدر صريحة فقط = 0 وهو `PYTHON_DERIVED`.

---

## SECTION 3 — FORMULA REGISTRY

المصدر: `REGISTRY = FormulaRegistry()` في `financial_engine/registry.py`  
الكتالوج: `FORMULAS` في `financial_engine/formulas.py`  
العدد الفعلي: **22**

البوابة: أي `formula_id` غير موجود يرفع `UnconfirmedFormulaError`  
الرسالة الفعلية تبدأ بـ: `UNREGISTERED FORMULA`

الحقول التي يعيدها `registry_entry()` فعلياً:

- `formula_id`
- `formula_name`
- `formula_expression`
- `required_inputs`
- `source_fields`
- `output_field`
- `aggregation_method`
- `validation_rule`

حقول إضافية على كائن `Formula` (موجودة، ليست في `registry_entry` إلا عبر خصائص أخرى):

- `grouping` = `Formula.group_by` (`Product` / `Country` / `None`)
- `sheet`
- `master_section`
- `identity_of` (لـ GETPIVOTDATA فقط)
- `operation`

`validation_rule` نص موحّد لكل المعادلات (يختلف فقط ذكر قسم MASTER):

> Execute only if formula_id is in Formula Registry. Use SOURCE cells only; never coerce MISSING to ZERO. Empty group or absent column → NOT AVAILABLE. Mixed or unknown currency → NOT AVAILABLE (no conversion). Denominator 0 for percent-of-total → NOT AVAILABLE. MASTER citation: …

حقول المصدر التي يطلبها السجل فعلياً:

`COGS`, `Sale Price`, `Manufacturing Price`, `Units Sold`, `Net Sales`, `Profit`, `Discounts`

أبعاد التجميع المستخدمة: `Product`, `Country`

### المعادلات المسجّلة (22)

#### F-COGS-BY-PRODUCT

- formula_name: COGS by Product
- formula_expression: `SUM(Table1[COGS]) GROUP BY Table1[Product]`
- required_inputs: COGS, Product
- source_fields: COGS
- output_field: COGS by Product
- aggregation_method: SUM
- grouping: Product
- sheet: COGS
- master_section: §4 COGS — منطق التجميع: COGS → SUM

#### F-COGS-GRAND-TOTAL

- formula_name: COGS Grand Total
- formula_expression: `SUM(Table1[COGS])`
- required_inputs: COGS
- source_fields: COGS
- output_field: COGS Grand Total
- aggregation_method: SUM
- grouping: None
- sheet: COGS
- master_section: §4 COGS — Grand Total

#### F-AVG-SALE-PRICE-BY-PRODUCT

- formula_name: Average Sale Price by Product
- formula_expression: `AVERAGE(Table1[Sale Price]) GROUP BY Table1[Product]`
- required_inputs: Sale Price, Product
- source_fields: Sale Price
- output_field: Average Sale Price by Product
- aggregation_method: AVERAGE
- grouping: Product
- sheet: Manufacturing Price,Sales
- master_section: §5 Manufacturing Price,Sales — Sale Price → Average

#### F-MAX-MFG-PRICE-BY-PRODUCT

- formula_name: Manufacturing Price by Product
- formula_expression: `MAX(Table1[Manufacturing Price]) GROUP BY Table1[Product]`
- required_inputs: Manufacturing Price, Product
- source_fields: Manufacturing Price
- output_field: Manufacturing Price by Product
- aggregation_method: MAX
- grouping: Product
- sheet: Manufacturing Price,Sales
- master_section: §5 Manufacturing Price,Sales — Manufacturing Price → Max

#### F-UNITS-SOLD-BY-PRODUCT

- formula_name: Units Sold by Product
- formula_expression: `SUM(Table1[Units Sold]) GROUP BY Table1[Product]`
- required_inputs: Units Sold, Product
- source_fields: Units Sold
- output_field: Units Sold by Product
- aggregation_method: SUM
- grouping: Product
- sheet: Unites Sold
- master_section: §6 Unites Sold / §10 Pivot 6

#### F-UNITS-SOLD-GRAND-TOTAL

- formula_name: Units Sold Grand Total
- formula_expression: `SUM(Table1[Units Sold])`
- required_inputs: Units Sold
- source_fields: Units Sold
- output_field: Units Sold Grand Total
- aggregation_method: SUM
- grouping: None
- sheet: Unites Sold
- master_section: §6 Unites Sold — Grand Total

#### F-UNITS-SOLD-PCT-BY-PRODUCT

- formula_name: Units Sold Percent of Total by Product
- formula_expression: `SUM(Table1[Units Sold] WHERE Product=p) / SUM(Table1[Units Sold])`
- required_inputs: Units Sold, Product
- source_fields: Units Sold
- output_field: Units Sold Percent of Total by Product
- aggregation_method: PERCENT_OF_TOTAL
- grouping: Product
- sheet: Unites Sold
- master_section: §6 Unites Sold — Units Sold → Percent of Total
- وحدة الناتج الفعلية: `ratio` (0–1) وليست نسبة مئوية 0–100

#### F-UNITS-SOLD-PCT-GRAND-TOTAL

- formula_name: Units Sold Percent of Total Grand Total
- formula_expression: `SUM(Table1[Units Sold]) / SUM(Table1[Units Sold])`
- required_inputs: Units Sold
- source_fields: Units Sold
- output_field: Units Sold Percent of Total Grand Total
- aggregation_method: PERCENT_OF_TOTAL
- grouping: None (مفتاح الناتج يستخدم التسمية `Grand Total`)
- إذا المقام = 0 أو غير متاح: NOT AVAILABLE (`DENOMINATOR_ZERO` / `DENOMINATOR_MISSING`)
- إذا المقام ≠ 0: القيمة `1`

#### F-NET-SALES-BY-COUNTRY

- formula_name: Net Sales by Country
- formula_expression: `SUM(Table1[Net Sales]) GROUP BY Table1[Country]`
- required_inputs: Net Sales, Country
- source_fields: Net Sales
- output_field: Net Sales by Country
- aggregation_method: SUM
- grouping: Country
- sheet: Net Sales
- master_section: §7 Net Sales — Net Sales → SUM

#### F-NET-SALES-GRAND-TOTAL

- formula_name: Net Sales Grand Total
- formula_expression: `SUM(Table1[Net Sales])`
- required_inputs: Net Sales
- source_fields: Net Sales
- output_field: Net Sales Grand Total
- aggregation_method: SUM
- grouping: None
- sheet: Net Sales

#### F-PROFIT-BY-COUNTRY

- formula_name: Profit by Country
- formula_expression: `SUM(Table1[Profit]) GROUP BY Table1[Country]`
- required_inputs: Profit, Country
- source_fields: Profit
- output_field: Profit by Country
- aggregation_method: SUM
- grouping: Country
- sheet: Profit
- master_section: §9 Profit — Profit → SUM

#### F-PROFIT-GRAND-TOTAL

- formula_name: Profit Grand Total
- formula_expression: `SUM(Table1[Profit])`
- required_inputs: Profit
- source_fields: Profit
- output_field: Profit Grand Total
- aggregation_method: SUM
- grouping: None
- sheet: Profit

#### F-KPI-PROFIT

- formula_name: KPI Sum of Profit
- formula_expression: `SUM(Table1[Profit])`
- required_inputs: Profit
- source_fields: Profit
- output_field: KPI Sum of Profit
- aggregation_method: SUM
- grouping: None
- sheet: KPIs
- master_section: §8 KPI 1 — A3 Sum of Profit / A4 value

#### F-KPI-COGS

- formula_name: KPI Sum of COGS
- formula_expression: `SUM(Table1[COGS])`
- required_inputs: COGS
- source_fields: COGS
- output_field: KPI Sum of COGS
- aggregation_method: SUM
- grouping: None
- sheet: KPIs
- master_section: §8 KPI 2 — A8 Sum of COGS / A9 value

#### F-KPI-NET-SALES

- formula_name: KPI Sum of Net Sales
- formula_expression: `SUM(Table1[Net Sales])`
- required_inputs: Net Sales
- source_fields: Net Sales
- output_field: KPI Sum of Net Sales
- aggregation_method: SUM
- grouping: None
- sheet: KPIs
- master_section: §8 KPI 3 — A13 Sum of Net Sales / A14 value

#### F-KPI-UNITS-SOLD

- formula_name: KPI Sum of Units Sold
- formula_expression: `SUM(Table1[Units Sold])`
- required_inputs: Units Sold
- source_fields: Units Sold
- output_field: KPI Sum of Units Sold
- aggregation_method: SUM
- grouping: None
- sheet: KPIs
- master_section: §8 KPI 4 — A18 Sum of Units Sold / A19 value

#### F-KPI-DISCOUNTS

- formula_name: KPI Sum of Discounts
- formula_expression: `SUM(Table1[Discounts])`
- required_inputs: Discounts
- source_fields: Discounts
- output_field: KPI Sum of Discounts
- aggregation_method: SUM
- grouping: None
- sheet: KPIs
- master_section: §8 KPI 5 — A23 Sum of Discounts / A24 value

#### F-DASHBOARD-PROFIT

- formula_name: Dashboard Profit
- formula_expression: `GETPIVOTDATA(" Profit ",KPIs!$A$3) ≡ F-KPI-PROFIT`
- required_inputs: Profit, F-KPI-PROFIT
- source_fields: Profit
- output_field: Dashboard Profit
- aggregation_method: GETPIVOTDATA_IDENTITY
- identity_of: F-KPI-PROFIT
- Dashboard cell in MASTER: E6  
- المسافات داخل اسم حقل GETPIVOTDATA جزء من المواصفة.

#### F-DASHBOARD-COGS

- formula_expression: `GETPIVOTDATA(" COGS ",KPIs!$A$8) ≡ F-KPI-COGS`
- identity_of: F-KPI-COGS
- Dashboard cell: I6

#### F-DASHBOARD-NET-SALES

- formula_expression: `GETPIVOTDATA("  Net Sales ",KPIs!$A$13) ≡ F-KPI-NET-SALES`
- identity_of: F-KPI-NET-SALES
- Dashboard cell: M6
- اسم الحقل في Excel يحتوي مسافتين قبل Net.

#### F-DASHBOARD-UNITS-SOLD

- formula_expression: `GETPIVOTDATA(" Units Sold ",KPIs!$A$18) ≡ F-KPI-UNITS-SOLD`
- identity_of: F-KPI-UNITS-SOLD
- Dashboard cell: O5

#### F-DASHBOARD-DISCOUNTS

- formula_expression: `GETPIVOTDATA(" Discounts ",KPIs!$A$23) ≡ F-KPI-DISCOUNTS`
- identity_of: F-KPI-DISCOUNTS
- Dashboard cell: S6

### غير مسجّل (لن يُنفَّذ)

أي شيء خارج الجدول أعلاه، بما في ذلك صراحةً في `FORBIDDEN_CONSTRUCTS`:

Income Statement, Balance Sheet, Cash Flow, NPV, IRR, ROA, ROE, EBITDA, WACC, ROI, Margin%, Gross Margin, Contribution Margin

وكذلك غير المسجّل رغم وجود أعمدته كـ SOURCE مستقلة:

- `Profit = Net Sales − COGS`
- `Net Sales = Gross Sales − Discounts`
- `Gross Sales = Units Sold × Sale Price`

`Operation.SOURCE_PASSTHROUGH` موجود في التعداد ولا تستخدمه أي معادلة مسجّلة.

ورقة Manufacturing Price,Sales: **لا Grand Total** في التنفيذ (`MFG_SALES_HAS_GRAND_TOTAL = False`).

---

## SECTION 4 — CALCULATION ENGINE

الملف: `financial_engine/engine.py`  
الدخول البرمجي: `CalculationEngine(dataset).compute()`  
الكتالوج يُمرَّر فقط عبر `REGISTRY.all()` و `REGISTRY.get()`.

### المسار الفعلي لحساب واحدة

```
Frozen Table1 (SOURCE cells)
    → formula_id ∈ REGISTRY
        → مدخلات SOURCE قابلة للاستخدام
            → SUM | AVERAGE | MAX | PERCENT_OF_TOTAL | GETPIVOTDATA_IDENTITY
                → PYTHON_DERIVED + AuditRecord + log COMPUTE
        → وإلا
            → NOT AVAILABLE + reason + AuditRecord
    → formula_id ∉ REGISTRY
        → UnconfirmedFormulaError / REJECT / NOT AVAILABLE
```

GETPIVOTDATA: نسخ قيمة KPI بعد حسابها. ليست ملاحظة SOURCE جديدة.

### قواعد التجميع الفعلية

- SUM / AVERAGE / MAX تستخدم الخلايا `is_available_number` فقط.
- MISSING يُستبعد ولا يُصفر.
- ZERO الصريح يُحسب.
- عملات مختلطة في نفس التجميع → `CURRENCY_MIXED_NO_CONVERSION` → NOT AVAILABLE.
- عملة غير معروفة على الخلية → الخلية غير مستخدمة؛ إذا لم يبقَ رقم → غالباً `UNKNOWN_CURRENCY` أو `ALL_VALUES_MISSING` حسب النطاق.
- مجموعة بلا صفوف → `NO_ROWS_FOR_GROUP` أو `NO_DATASET_ROWS`.
- عمود غائب → `MISSING_COLUMN`.
- Grand Total = مجموع **كل** صفوف المصدر في النطاق، ليس مجموع خانات قائمة القالب فقط.
- خانات القالب المتوقعة بلا صفوف تُدرج `NOT AVAILABLE` (`NO_ROWS_FOR_GROUP`).
- منتجات/دول إضافية في المصدر تُحسب وتُعلَّم كقيمة مصدر إضافية.
- صفوف ببعد تجميع فارغ تُحسب تحت المفتاح `__BLANK__` وتدخل الإجمالي.

خانات القالب المتوقعة (ليست بيانات عميل؛ للعرض فقط):

- Products: Amarilla, Carretera, Montana, Paseo, Velo, VTT
- Countries: Canada, France, Germany, Mexico, United States of America

أوراق النتائج الرسمية التي يجمّعها المحرك:

`COGS` · `Manufacturing Price,Sales` · `Unites Sold` · `Net Sales` · `Profit` · `KPIs` · `DashBoard`

ورقة `Company Financials (Dataset)` ليست ورقة ناتج مشتق.

`FilterSpec` موجود. الافتراضي: بلا مرشح. المرشحات لا تُستنتج من `Slicer_*` / `NativeTimeline_Date`.

---

## SECTION 5 — VALIDATION ENGINE

الملف: `financial_engine/validation.py`  
الاستدعاء الفعلي في المسار الرسمي: **بعد** `compute()` من داخل `run_pipeline`.  
Validation **لا يوقف الحساب مسبقاً**. لا يوجد في التنفيذ باب validation-before-compute داخل `run_pipeline`.

الحالات:

- `VALIDATION CLEAN` — لا ERROR ولا FAIL
- `VALIDATION WARNING` — توجد WARN فقط؛ `is_clean == True`
- `VALIDATION FAILED` — يوجد ERROR أو FAIL؛ `is_clean == False`

CLI بعد كتابة التقارير: إذا `not is_clean` أو `not is_reconciled` → exit code **1**.  
الناتج يُكتب رغم ذلك.

ما يطبّقه المحرك فعلياً:

| الموضوع | السلوك الفعلي | الرموز |
|---|---|---|
| Required columns | تحذير إن غاب حقل MASTER أو حقل يحتاجه السجل. لا اختراع عمود | `MISSING_MASTER_FIELDS`, `REGISTRY_INPUT_ABSENT` |
| Data types | أرقام عبر `parse_numeric`؛ boolean ليس رقماً | `TEXT_IN_NUMERIC_FIELD`, `UNPARSEABLE_NUMERIC` |
| Missing values | جرد ZERO/MISSING/PRESENT/NOT_AVAILABLE. لا تحويل | `PRESENCE_INVENTORY` |
| Duplicates | تحذير. الصفوف لا تُحذف وتدخل SUM | `DUPLICATE_ROWS` |
| Dates | ISO `YYYY-MM-DD` فقط. غير ذلك تحذير. لا تجميع حسب التاريخ | `DATE_UNPARSEABLE` |
| Currency | عملة مجهولة/مختلطة: تحذير. لا تحويل | `UNKNOWN_CURRENCY` |
| Units | يُسجَّل أن MASTER لا يعرّف وحدة. لا تُفترض | `UNIT_NOT_ASSUMED` |
| Invalid / classification | سلامة الناتج وتصنيف ممنوع | `NA_HAS_VALUE`, `DERIVED_WITHOUT_FORMULA`, `NON_DECIMAL_VALUE`, … |
| Formula compatibility | السجل 22 معادلة؛ عمليات Pivot فقط؛ لا KPI إضافي | `REGISTRY_GATE`, `EXTRA_KPI`, `ILLEGAL_OPERATION` |
| Forbidden constructs | ظهور NPV/IRR/ROE… في المفاتيح/التسميات = FAIL | `FORBIDDEN_CONSTRUCT` |
| Source integrity | hash الحي = hash الحزمة | `SOURCE_FROZEN`, `SOURCE_MUTATED`, `SOURCE_HASH_DRIFT` |
| Extra headers | تُحفظ في المصدر وتُتجاهل رسمياً | `EXTRA_HEADERS` |

لا يوجد في التنفيذ فحص «قيمة غير منطقية» بمعنى `Profit ≟ Net Sales − COGS`.

---

## SECTION 6 — RECONCILIATION

الملف: `financial_engine/reconciliation.py`  
المسار الثاني: `IndependentAggregator` — لا يستدعي مجمّعات `CalculationEngine`.

**ما يُطابق:**

- SUM لكل معادلات SUM المسجّلة (حسب المجموعة والإجمالي وKPI)
- AVERAGE لـ Sale Price حسب Product
- MAX لـ Manufacturing Price حسب Product
- PERCENT_OF_TOTAL لوحدات مباعة + Grand Total
- هويات MASTER فقط:
  - `F-KPI-PROFIT` ≡ `F-PROFIT-GRAND-TOTAL` ≡ `F-DASHBOARD-PROFIT`
  - `F-KPI-COGS` ≡ `F-COGS-GRAND-TOTAL` ≡ `F-DASHBOARD-COGS`
  - `F-KPI-NET-SALES` ≡ `F-NET-SALES-GRAND-TOTAL` ≡ `F-DASHBOARD-NET-SALES`
  - `F-KPI-UNITS-SOLD` ≡ `F-UNITS-SOLD-GRAND-TOTAL` ≡ `F-DASHBOARD-UNITS-SOLD`
  - `F-KPI-DISCOUNTS` ≡ `F-DASHBOARD-DISCOUNTS`
- مجموع نسب المنتجات PYTHON_DERIVED = 1

**نجاح:** `status == "RECONCILED"` و `mismatch_count == 0`

**فشل:** `status == "RECONCILIATION FAILED"` عند أي اختلاف قيمة أو توفر بين المسارين، أو كسر هوية KPI/Dashboard/Grand Total، أو مجموع النسب ≠ 1.

لا يُخفى الاختلاف.

لا يطابق هويات صفّية غير موجودة في MASTER.

Grand Total مقابل مجموع خانات القالب فقط: يُسجَّل تحذير `GRAND_TOTAL_SCOPE` وليس فشلاً.

---

## SECTION 7 — SOURCE INTEGRITY

الملفات: `dataset.py`, `integrity.py`, `engine.py`

**Checksum**

- `FrozenDataset.source_hash`: SHA-256 للقطة المصدر عند التجميد (الحقول + الصفوف + id + origin).
- `source_checksum` لكل ناتج مشتق: SHA-256 للخلايا المستخدمة في ذلك الناتج.

**Read only**

- صف المصدر `_FrozenMap`: أي كتابة ترفع `SourceMutationError`.
- `assert_unchanged(expected_hash)` يعيد حساب الهاش.

**Mutation detection**

- `CalculationEngine.verify_source_integrity()` بعد `compute` ومرة أخرى في نهاية `run_pipeline`.
- عند الاختلاف: يسجّل الحدث `SOURCE_MUTATION_DETECTED` ويرفع `SourceMutationDetected("SOURCE MUTATION DETECTED. Calculation halted.")`.
- CLI exit code **5**.

**Fixture ≠ CLIENT DATA**

علامات الرفض في الإنتاج (`ProductionFixtureError`, exit **4**):

`tests/fixtures`, `tests\fixtures`, `test:`, `fixture:`, `tiny_balanced`

التجاوز الوحيد: `allow_fixture=True` / `--allow-fixture`.

---

## SECTION 8 — AUDIT TRAIL

### AuditRecord (يُنشأ في `_log_result`)

حقول فعلية:

| الحقل | موجود |
|---|---|
| result_type | نعم (`SOURCE` / `PYTHON_DERIVED` / `NOT_AVAILABLE`) |
| formula_id | نعم |
| formula_expression | نعم |
| source_fields | نعم |
| aggregation | نعم |
| group_by | نعم |
| source_checksum | نعم |
| result | نعم (نص عشري أو null) |
| computed_at_utc | نعم |
| n_source_rows_used | نعم |
| source_cells | نعم (مراجع Excel، قد تُقطع عند 50) |
| inputs | نعم (`group_value`, `n_missing`, `n_unparseable`, `n_in_scope`) |
| log_entry_id | نعم |
| reason | نعم |

يُكتب إلى `audit_trail.json` عبر `write_all_reports`.

`ResultBundle.audit_records`: تُعيَّن ديناميكياً في `compute()`. ليست حقلاً معلناً في dataclass. `to_dict()` يقرأها بعد التعيين.

### CalculationLog.LogEntry

`id`, `timestamp_utc`, `event`, `message`, `formula_id`, `result_key`, `inputs`, `output_classification`, `output_value`, `reason`

أحداث مستخدمة فعلياً تشمل: `LOAD`, `CLASSIFY`, `FILTER`, `COMPUTE`, `REJECT`, `VALIDATE`, `RECONCILE`, `SOURCE_MUTATION_DETECTED`.

---

## SECTION 9 — TEST RESULTS

آخر تشغيل مسجّل في بيئة البناء الحالية:

- **عدد الاختبارات:** 76
- **الناجحة:** 76
- **الفاشلة:** 0

نجاح الاختبارات ≠ شهادة على بيانات عميل. لا يوجد CLIENT DATA حقيقي في مرحلة 2.

ملفات الاختبار الموجودة:

- `tests/test_classification.py`
- `tests/test_engine.py`
- `tests/test_no_invention.py`
- `tests/test_traceability.py`
- `tests/test_validation_and_reconciliation.py`
- `tests/test_reproducibility.py`
- `tests/test_schema_and_formulas.py`
- `tests/test_audit_and_loaders.py`
- `tests/test_stage2_mandatory.py` (TEST A–J)

TEST A–J كما هي موثّقة في `tests/TEST_CATALOG.md`:

| ID | الموضوع |
|---|---|
| A | بيانات كاملة صحيحة (fixture) |
| B | بيانات ناقصة |
| C | نص في حقل مالي |
| D | تكرارات |
| E | فراغ مقابل صفر صريح |
| F | عملة غير معروفة |
| G | عمود غير موجود |
| H | معادلة غير مسجّلة |
| I | تغيير المصدر بعد الحساب |
| J | اختلاف Python عن المرجع المستقل |

بيانات `tests/fixtures/` Fixture فقط.

---

## SECTION 10 — ENGINE LIMITATIONS

مستخرجة من التنفيذ و`docs/ENGINE_LIMITS.md`:

1. لا CLIENT DATA حقيقي داخل حزمة المرحلة 2. لا اعتماد إنتاجي على ناتج شركة.
2. المعادلات غير المسجّلة مرفوضة. لا هامش/NPV/IRR/ROA/ROE/EBITDA/قوائم مالية.
3. أعمدة Gross Sales وSegment وDiscount Band وDate وMonth وYear موجودة في المخطط كـ SOURCE إن وُجدت، لكن **لا معادلة رسمية تجمّعها** ما عدا ما في السجل.
4. Date / Timeline / Slicers: لا تجميع شهري/ربعي رسمي. بعض أسماء القالب تشير إلى `#N/A` في MASTER §18. المرشحات لا تُستنتج.
5. التواريخ غير ISO تُحذَّر ولا تُفسَّر.
6. العملة: لا تحويل. مختلط/مجهول → لا تجميع.
7. الوحدة: NOT DEFINED في MASTER. المحرك يضع `unit=None` إلا لنسب الوحدات (`ratio`).
8. خانة قالب بلا صفوف مصدر = NOT AVAILABLE. المرحلة 3 لا تكتب 0.
9. اسم الورقة الرسمي: `Unites Sold` (إملاء القالب). لا يُصحَّح.
10. XLSX القالب الأصلي قد يفشل openpyxl بسبب XML الرسوم (MASTER §18). `LoadError`. لا صفوف مخترعة.
11. `#NAME?` في Dashboard القالب ليس رقماً.
12. بيانات 2013–2014 في القالب وصف للعيّنة الحالية لا قيد حسابي.
13. `ResultBundle.engine_version` الافتراضي `1.0.0` بينما `__version__` = `2.0.0`.
14. Validation بعد الحساب لا يمنع إنتاج الملفات.
15. حقن Master Template: **NOT DEFINED في المرحلة 2**.
16. Pivot Tables المادية داخل xlsx: المحرك يعيد منطق التجميع ولا يعيد بناء كائنات Pivot في Excel.
17. Dashboard charts: خارج نطاق المحرك.

---

## SECTION 11 — STAGE 3 INPUT CONTRACT

المدخل الوحيد للأرقام: **CLIENT DATA** كجدول Table1.

المحرّك يقبل عبر `load_table1`:

- `.csv` / `.tsv` / `.txt`
- `.json` (قائمة كائنات، أو مفتاح `rows` / `data` / `Table1` / `records`)
- `.xlsx` / `.xlsm` (`data_only=True`)
- قائمة dict في الذاكرة

أسماء أوراق Excel التي يبحث عنها بالترتيب إن لم يُمرَّر `--sheet`:

1. `Company Financials (Dataset)`
2. `Company Financials`
3. `Dataset`
4. `Table1`
5. الورقة الوحيدة إن وُجدت ورقة واحدة فقط

إن التبس الاسم: `LoadError`. لا تخمين.

الحقول الستة عشر (MASTER §3). الأسماء بعد `canonicalize_header`:

| العمود | الاسم | الدور |
|---|---|---|
| A | Segment | dimension |
| B | Country | dimension |
| C | Product | dimension |
| D | Discount Band | dimension |
| E | Units Sold | numeric |
| F | Manufacturing Price | numeric |
| G | Sale Price | numeric |
| H | Gross Sales | numeric |
| I | Discounts | numeric |
| J | Net Sales | numeric |
| K | COGS | numeric |
| L | Profit | numeric |
| M | Date | date |
| N | Month Number | calendar |
| O | Month Name | calendar |
| P | Year | calendar |

حقل غير معروف يبقى extra header ولا يصبح معادلة.

**قبل الحساب على المرحلة 3:**

1. فهم CLIENT_DATA.xlsx وتمييزه عن أي أرقام داخل MASTER_ARABIC_TEMPLATE.xlsx.
2. تعيين أعمدة العميل إلى أسماء Table1 أعلاه فقط. لا اختراع حقل.
3. إنتاج ملف/سجلات Table1 وتمريرها بـ `load_table1` أو `run_pipeline`.
4. عدم تمرير مسار/معرف يشبه Fixture إلا باختبار معلن.
5. عدم ملء الفراغ بأصفار أو بعيّنة القالب.

واجهة الاستدعاء الرسمية الموجودة:

```python
from financial_engine import load_table1, run_pipeline

dataset = load_table1("mapped_table1.csv")  # CLIENT DATA بعد الربط
result = run_pipeline(dataset)
# result.bundle / result.validation / result.reconciliation / result.log
```

حقن الخلايا في القالب العربي: **NOT DEFINED في المرحلة 2**. مسؤولية المرحلة 3 بعد قراءة `result.bundle`.

---

## SECTION 12 — STAGE 3 RESPONSIBILITIES

المرحلة 3 مسؤولة عن:

```
CLIENT DATA
  → DATA UNDERSTANDING
    → DATA MAPPING  (إلى أسماء Table1 فقط)
      → ENGINE INPUT
        → run_pipeline / CalculationEngine
          → قراءة PYTHON_DERIVED / SOURCE / NOT AVAILABLE
            → MASTER TEMPLATE INJECTION
```

المرحلة 3 ليست مسؤولة عن اختراع بيانات أو معادلات أو تعديل Registry أو اعتبار أرقام القالب بيانات عميل.

---

## SECTION 13 — MASTER TEMPLATE INTERFACE

`MASTER_ARABIC_TEMPLATE.xlsx` (عند توفره للمرحلة 3):

- قالب عرض وبنية (8 أوراق، Pivot، KPI cards، Charts، GETPIVOTDATA).
- **ليس** مصدر CLIENT DATA.
- أي رقم نموذجي داخل القالب **ليس** بيانات عميل.
- المحرك الحالي لا يفتح هذا القالب ولا يكتب فيه.
- المرحلة 3 تحقن فقط نتائج المحرك في المواضع التي يعرّفها MASTER.
- إن كانت نتيجة المحرك `NOT AVAILABLE` تُترك غير رقمية. لا تُستبدل بعيّنة القالب ولا بـ `#NAME?` ولا بـ 0.
- الحفاظ على: عدد الأوراق، الترتيب، أسماء الأوراق ما لم توجد آلية تعريب لا تكسر العلاقات، Table1، Pivot logic، مصادر الرسوم، علاقات GETPIVOTDATA.

خصائص القالب المسجّلة في MASTER §18 (ليست أخطاء يصلحها المحرك بأرقام):

- بعض GETPIVOTDATA تظهر `#NAME?`
- XML الرسوم قد يكسر openpyxl
- 10 Pivot Tables على Cache 0 من Table1
- 10 Charts
- `NativeTimeline_Date`, `Slicer_Country`, `Slicer_Product`, `Slicer_Segment` قد تشير إلى `#N/A`

---

## SECTION 14 — NOT AVAILABLE HANDLING

إذا تعذّر الحساب بالمعادلة المسجّلة والمدخل SOURCE:

`classification = NOT_AVAILABLE`  
`value = None`

أسباب مستخدمة فعلياً (`AvailabilityReason`):

`MISSING_COLUMN` · `NO_DATASET_ROWS` · `NO_ROWS_FOR_GROUP` · `ALL_VALUES_MISSING` · `UNPARSEABLE_VALUE` · `DENOMINATOR_MISSING` · `DENOMINATOR_ZERO` · `CURRENCY_MIXED_NO_CONVERSION` · `UNCONFIRMED_FORMULA` · `UPSTREAM_NOT_AVAILABLE` · `FILTER_EXCLUDES_ALL` · `UNKNOWN_CURRENCY` · `SOURCE_MUTATION_DETECTED` · `UNREGISTERED_FORMULA`

المرحلة 3:

- لا تستخدم `0`
- لا تستخدم Template Sample Data
- لا تستخدم Guess
- لا تستخدم `#NAME?` كرقم

---

## SECTION 15 — ERROR CONDITIONS

| الحالة | المصدر | أثر التنفيذ |
|---|---|---|
| `LOAD FAILED` / `LoadError` | ملف مفقود / JSON غير صالح / Excel غير قابل للفتح / ورقة غير معيّنة | CLI exit **2**. لا صفوف مخترعة |
| `PRODUCTION BOUNDARY` / `ProductionFixtureError` | Fixture قُدّم كعميل | CLI exit **4** |
| `UNREGISTERED FORMULA` / `UnconfirmedFormulaError` | معادلة خارج السجل | CLI exit **3**؛ أو NOT AVAILABLE عبر `reject_unconfirmed` |
| `SOURCE MUTATION DETECTED` / `SourceMutationDetected` | تغيّر الهاش بعد التجميد | إيقاف. CLI exit **5** |
| `SourceMutationError` | كتابة على صف مجمّد | استثناء فوري |
| `VALIDATION FAILED` | ERROR/FAIL في التقرير | الملفات تُكتب. CLI exit **1** |
| `RECONCILIATION FAILED` | mismatch | الملفات تُكتب. CLI exit **1** |
| `VALIDATION WARNING` | WARN فقط | لا يفشل `is_clean`. CLI 0 ما لم تفشل المطابقة |
| Mixed/unknown currency | تجميع | NOT AVAILABLE للناتج المعني. لا تحويل |

`SchemaError`, `ClassificationError`, `CurrencyPolicyError` معرّفة. استخدامها كباب CLI مستقل: **NOT DEFINED** (لا exit code خاص بها في `__main__.py`).

---

## SECTION 16 — REQUIRED EXECUTION ORDER

الترتيب الفعلي في `run_pipeline` + `__main__` (وليس نص الـ docstring وحده):

```
CLIENT DATA file
  → refuse_fixture_as_client
  → load_table1 / freeze_records     # تجميد + source_hash
  → CalculationEngine.compute()      # Registry فقط + Audit أثناء الحساب
  → verify_source_integrity()
  → ValidationEngine.validate()      # المصدر + النتائج معاً
  → ReconciliationEngine.reconcile()
  → verify_source_integrity()
  → write_all_reports()              # بما فيه audit_trail.json
  → MASTER INJECTION                 # NOT DEFINED في المرحلة 2 — مرحلة 3
```

فرق عن العبارة «VALIDATION ثم CALCULATION»:  
**الحساب يتم أولاً، ثم Validation، ثم Reconciliation.**  
Docstring في `pipeline.py` يذكر «validate source → compute» لكن جسم الدالة لا يستدعي Validation قبل `compute()`.

---

## SECTION 17 — DO NOT IMPLEMENT

المرحلة 3 ممنوعة من:

- Invent data
- Guess missing values
- Add formulas
- Modify source
- Treat template values as client data
- Change currency without authorization (المحرك يرفض التحويل أصلاً)
- Replace missing with zero
- Modify registered formulas
- Create unsupported KPIs
- Rebuild the engine
- Redesign Dashboard
- Reorder or drop MASTER sheets/charts/pivots
- Infer slicer/timeline filters
- Pass tests/fixtures as CLIENT DATA
- Fill template product/country slots with 0 when engine says NOT AVAILABLE
- Use `Profit = Net Sales − COGS` or any unregistered identity
- Silent correction of client numbers

---

## SECTION 18 — HANDOFF EXAMPLE

مثال تجريدي. ليست بيانات عميل حقيقية.

**1. CLIENT DATA (مفهومياً بعد الفهم)**  
صفوف مبيعات فيها على الأقل الحقول التي ستُربط إلى Table1.

**2. MAPPING**  
عمود العميل «صافي المبيعات» → `Net Sales`.  
عمود غير موجود في المخطط → extra أو يُترك. لا يُخلق KPI منه.

**3. ENGINE**

```text
load_table1(mapped_table)
run_pipeline(dataset)
```

**4. OUTPUT CLASSES**

- خلية `Profit = 150` في Table1 → SOURCE (إن بقيت كما هي في المصدر)
- `F-KPI-PROFIT = SUM(Profit)` → PYTHON_DERIVED + checksum + lineage إلى `Table1!L…`
- منتج في قائمة القالب بلا صفوف → `F-COGS-BY-PRODUCT::<name>` = NOT AVAILABLE
- خلية Profit فارغة → MISSING؛ لا تدخل المجموع كصفر
- خلية Profit مكتوب فيها 0 → ZERO وتدخل المجموع

**5. MASTER TEMPLATE**  
المرحلة 3 تضع PYTHON_DERIVED في خانة KPI المطابقة.  
تترك NOT AVAILABLE بلا رقم.  
لا تنسخ أرقام عيّنة القالب.

---

## SECTION 19 — COPILOT INTEGRATION CONTRACT

Copilot في المرحلة 3 يستلم:

1. `MASTER_ARABIC_TEMPLATE.xlsx`
2. `CLIENT_DATA.xlsx`
3. `STAGE_2_HANDOFF_SPEC.md`

يجب أن يستخدم هذا الملف كمواصفة تشغيل للمرحلة 2.

يلتزم:

- لا يعيد بناء المحرك.
- لا يخترع معادلات.
- لا يغيّر Master.
- لا يعتبر أرقام Master بيانات العميل.
- يمرّر إلى المحرك Table1 بعد mapping فقط.
- يقرأ `result.bundle` / التقارير ولا يستبدل NOT AVAILABLE.
- يستدعي `run_pipeline` أو `CalculationEngine` كما هما.
- يتوقف عند `SOURCE MUTATION DETECTED` ولا يكمل الحقن كنجاح.
- إن `RECONCILIATION FAILED` أو `VALIDATION FAILED`: لا يُعلن اكتمال الحقن.

نقطة الدخول الموثّقة للحزمة: `run_stage2.py` و `python -m financial_engine`.

---

## SECTION 20 — FINAL HANDOFF STATUS

### STAGE 2 HANDOFF STATUS

**ENGINE:**  
منفَّذ. يحسب 22 معادلة مسجّلة فقط من Table1 المجمّد. لا حقن Excel.

**VALIDATION:**  
منفَّذ بعد الحساب. حالات CLEAN / WARNING / FAILED. لا يمنع `compute()`.

**RECONCILIATION:**  
منفَّذ بمسار مستقل. RECONCILED / RECONCILIATION FAILED.

**AUDIT:**  
منفَّذ (`AuditRecord` + Calculation Log + `audit_trail.json`).

**FORMULA REGISTRY:**  
مغلق. 22 معادلة. غير المسجّل مرفوض.

**TESTS:**  
76 / 76 / 0 على Fixtures. ليست شهادة CLIENT DATA.

**KNOWN LIMITATIONS:**  
انظر SECTION 10. أبرزها: لا CLIENT DATA حقيقي؛ لا حقن قالب؛ لا تجميع زمني رسمي؛ لا تحويل عملة؛ لا افتراض وحدة؛ NOT AVAILABLE ≠ 0؛ اختلاف `__version__` 2.0.0 عن `ResultBundle.engine_version` 1.0.0؛ Validation بعد الحساب.

**STAGE 3 HANDOFF:**  
**READY WITH LIMITATIONS**

ليست READY بلا قيد: لا بيانات عميل حقيقية للاعتماد، وحقن القالب غير موجود في المرحلة 2، وقيود التاريخ/العملة/الوحدة/خانات القالب قائمة ويجب أن تلتزم بها المرحلة 3 حرفياً.
