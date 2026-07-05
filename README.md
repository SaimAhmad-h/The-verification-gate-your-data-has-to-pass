# DataGate

### E-Commerce Order Data Cleaning & Verification Pipeline

A Python/pandas pipeline that audits, cleans, and formally verifies a raw e-commerce order
dataset. Built as **Project 1: Data Cleaning & Preparation** for the DecodeLabs Industrial
Training Kit, Batch 2026.

---

## Table of Contents

- [Overview](#overview)
- [Why This Project Exists](#why-this-project-exists)
- [Honest Note on the Dataset Used](#honest-note-on-the-dataset-used)
- [Dataset Schema](#dataset-schema)
- [Pipeline Stages in Detail](#pipeline-stages-in-detail)
- [The Verification Gate](#the-verification-gate)
- [Sample Change Log Output](#sample-change-log-output)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [How to Run](#how-to-run)
- [How to Run This on Your Own Dataset](#how-to-run-this-on-your-own-dataset)
- [Design Decisions & Reasoning](#design-decisions--reasoning)
- [Limitations](#limitations)
- [Possible Future Improvements](#possible-future-improvements)
- [License](#license)
- [Credits](#credits)

---

## Overview

This is **not** a dashboard, machine learning model, or analytics/visualization project.
It is a **data-quality pipeline** — the unglamorous but essential first step of any real
analytics workflow. Given a raw `.xlsx` order dataset, the pipeline:

1. **Audits** the dataset for missing values, duplicate records, and formatting issues
2. **Fixes** what it finds — imputation, deduplication, standardization
3. **Verifies**, with hard code-level assertions (not just a claim in a report), that the
   output meets a defined quality bar
4. **Documents** every single modification in an auditable, timestamped change log
   (Excel + formatted PDF)

The entire pipeline runs inside a single Jupyter notebook, `Project1_Data_Cleaning.ipynb`,
broken into clearly separated, independently readable cells — each cell does one job.

---

## Why This Project Exists

Industry data is rarely analysis-ready. Bad data that goes unnoticed doesn't throw an
error — it just quietly produces wrong totals, wrong charts, and wrong conclusions.

This project treats data cleaning as a discipline with its own standards, not a one-off script:
- Every fix is **justified and logged**, not silently applied
- The "is this actually clean now?" question is **answered with code**, not assumed
- Fixing issues (blank fields, duplicate keys, bad dates) uses defensible statistical/logical
  methods (median imputation, canonical deduplication, ISO 8601 standardization) rather than
  ad hoc guesses

---

## Honest Note on the Dataset Used

The specific dataset processed in this repository (`Dataset_for_Data_Analytics.xlsx` — 1,200
rows, 14 columns of e-commerce order data) turned out to be **largely clean already** when
audited. Full transparency on what was actually found:

| Issue category | Found in this dataset? |
|---|---|
| Missing values | Yes — 309 blank `CouponCode` values (25.75% of rows) |
| Full-row duplicates | None found |
| Duplicate `OrderID` values | None found |
| Duplicate `TrackingNumber` values | None found |
| Inconsistent date formats | None found — dates parsed cleanly |
| Whitespace / casing inconsistencies | None found |
| `TotalPrice ≠ Quantity × UnitPrice` mismatches | None found |

The `CouponCode` nulls are not really "dirty data" either — a blank there legitimately means
"no coupon was applied to this order," not a data entry error.

**This README will not overstate what the pipeline "rescued."** The code itself is written
generically and defensively (see [Design Decisions](#design-decisions--reasoning) below) so
that it will actively detect and repair a genuinely messy dataset — duplicate records, mixed
date formats, inconsistent capitalization, incorrect totals — if pointed at one. The clean
result in this repo's output files simply reflects that this particular dataset started in
good shape. The value of the project is in the *pipeline and its verification discipline*,
not in a dramatic before/after on this specific file.

---

## Dataset Schema

| Column | Type | Description |
|---|---|---|
| `OrderID` | string | Unique order identifier (e.g. `ORD200000`) |
| `Date` | date | Order date |
| `CustomerID` | string | Customer identifier (e.g. `C72649`) |
| `Product` | string | Product name (Monitor, Phone, Tablet, Chair, Printer, Laptop, Desk) |
| `Quantity` | integer | Units ordered |
| `UnitPrice` | float | Price per unit |
| `ShippingAddress` | string | Shipping street address |
| `PaymentMethod` | string | Debit Card, Credit Card, Online, Gift Card, Cash |
| `OrderStatus` | string | Shipped, Cancelled, Returned, Delivered, Pending |
| `TrackingNumber` | string | Shipment tracking code |
| `ItemsInCart` | integer | Number of items in the cart at checkout |
| `CouponCode` | string / null | Coupon applied, if any |
| `ReferralSource` | string | Instagram, Referral, Email, Facebook, Google |
| `TotalPrice` | float | `Quantity x UnitPrice` |

---

## Pipeline Stages in Detail

### 1. Missing Value Audit
Counts nulls per column and reports both raw count and percentage before any changes are made.

### 2. Imputation (never blind deletion)
Listwise deletion (dropping any row with a gap) reduces statistical power and can bias results.
Instead:
- **Numeric columns** → filled with the **median** (robust to outliers, unlike the mean)
- **Categorical/text columns** → filled with the **mode** (most frequent value)
- **`CouponCode` specifically** → filled with an explicit `"No Coupon"` label, since a blank
  there is a legitimate business state, not an error

### 3. Duplicate Audit & Removal
- Checks for fully duplicated rows
- Checks for duplicate `OrderID` values using a `groupby`-based check equivalent to
  `GROUP BY OrderID HAVING COUNT(*) > 1` in SQL
- Removes exact duplicate rows outright
- Collapses duplicate `OrderID`s down to a single canonical record (first occurrence kept)

### 4. Text Standardization
- Strips leading/trailing whitespace from every text column
- Applies consistent Title Case to categorical fields (e.g. `"credit card"` → `"Credit Card"`)
- Skips ID-style columns (`OrderID`, `CustomerID`, `TrackingNumber`, `CouponCode`) since
  altering their casing would be incorrect

### 5. Date Standardization
- Parses the `Date` column using `pandas.to_datetime`, which handles multiple input formats
- Any value that cannot be parsed becomes an explicit null (`NaT`) rather than a silent
  wrong guess
- Reformats every valid date into **ISO 8601** (`YYYY-MM-DD`) — unambiguous, sortable, and
  usable in date arithmetic

### 6. Numeric Precision
Rounds monetary columns (`UnitPrice`, `TotalPrice`) to exactly 2 decimal places, removing
floating-point noise (e.g. `570.6199999999999` → `570.62`).

### 7. Business-Logic Consistency Check
Beyond formatting, verifies the data is *factually* correct: recalculates
`Quantity x UnitPrice` for every row and compares it to the stored `TotalPrice`. Any row
more than one cent off is corrected. This catches arithmetic errors that would otherwise be
invisible — a row can look perfectly formatted and still contain the wrong number.

### 8. Documentation (Change Log)
Every fix applied by stages 2–7 is logged with:
- A **Change ID** (e.g. `CR001`, `CR_DEDUPE`)
- A plain-English **Description** of what was done
- The **Impact** (how many records/cells were affected)
- A **Status** (Resolved)

This list is exported both as a spreadsheet (`Change_Log.xlsx`) and a formatted PDF report
(`Change_Log.pdf`) with a styled, word-wrapped table suitable for handing to a stakeholder.

---

## The Verification Gate

The final pipeline stage does not fix anything — it **proves** the earlier stages worked, and
refuses to let the notebook proceed silently if they didn't:

```python
assert dup_id_rate == 0,   "Verification Gate failed: duplicate OrderIDs remain"
assert bad_date_rate == 0, "Verification Gate failed: badly formatted dates remain"
assert remaining_nulls == 0, "Verification Gate failed: missing values remain"
```

If any assertion fails, the notebook **stops with an error** instead of quietly producing an
export that still has problems. A completed run is verifiable evidence, not just an assumption.

Required thresholds (from the project brief):
- **0% duplicate unique identifiers**
- **0% incorrectly formatted dates**

---

## Sample Change Log Output

From an actual run against the included dataset:

| Change ID | Description | Impact | Status |
|---|---|---|---|
| CR001 | Imputed `CouponCode` missing values with explicit label `"No Coupon"` | Preserved 309 records (avoided deletion) | Resolved |
| CR_DEDUPE | Removed full-row duplicates and collapsed duplicate `OrderID` records | Removed 0 duplicate record(s); 1,200 unique records remain | Resolved |
| CR_TEXT_STD | Trimmed whitespace and applied consistent Title Case | Corrected formatting on 0 cell(s) | Resolved |
| CR_DATE_ISO | Converted `Date` column to ISO 8601 format | Standardized 1,200 records; 0 unparseable dates | Resolved |
| CR_NUMERIC_PRECISION | Rounded `UnitPrice`, `TotalPrice` to 2 decimal places | Applied to 1,200 records | Resolved |
| CR_LOGIC_TOTALPRICE | Verified `TotalPrice` equals `Quantity x UnitPrice` | No corrections needed — 0 mismatches found | Resolved |

**Verification Gate result:** Duplicate ID error rate = 0.00% · Bad date format rate = 0.00% · Remaining nulls = 0 → **PASSED**

---


---

## Requirements

```
pandas
numpy
openpyxl
reportlab
```

Install everything with:
```bash
pip install pandas numpy openpyxl reportlab
```

---

## How to Run

1. Clone this repository or download its files into one folder.
2. Make sure `Dataset_for_Data_Analytics.xlsx` is in the same folder as the notebook
   (or update the `RAW_FILE` path inside Cell 2 to point to your file's location).
3. Open `Project1_Data_Cleaning.ipynb` in Jupyter Notebook, JupyterLab, or VS Code.
4. Run every cell from top to bottom (`Kernel → Restart & Run All` is the safest option).
5. If the Verification Gate cell raises an `AssertionError`, the dataset still has an
   unresolved issue that needs a look — that is the pipeline correctly refusing to certify
   imperfect data as clean, not a bug.
6. On success, three files are written to the project folder:
   `Cleaned_Dataset_for_Data_Analytics.xlsx`, `Change_Log.xlsx`, and `Change_Log.pdf`.

---

## How to Run This on Your Own Dataset

The pipeline is written generically, so it should work on any similarly structured
order/transaction dataset with minimal changes:

1. Update `RAW_FILE` in Cell 2 to your file's path.
2. Check that your column names match what the notebook expects (`OrderID`, `Date`,
   `Quantity`, `UnitPrice`, `TotalPrice`, etc.), or adjust the column references in
   Cells 5–11 to match your schema.
3. Re-run all cells. The pipeline will report exactly what it found and fixed for *your*
   data — including cases with real duplicates, mixed date formats, or arithmetic errors,
   which this particular sample dataset did not have.

---

## Design Decisions & Reasoning

- **Median over mean for numeric imputation** — the median is far less sensitive to outliers
  (e.g. one $10,000 order won't drag every imputed value upward).
- **Explicit "No Coupon" label instead of imputing a fake coupon code** — imputing a
  categorical placeholder value that doesn't exist in the real category set would be
  misleading; a clear label preserves meaning.
- **Deduplicate by keeping the first occurrence** — a simple, defensible, reproducible rule.
  (An alternative would be keeping the *most complete* row, which is a reasonable extension
  — see [Limitations](#limitations).)
- **`errors="coerce"` on date parsing** — an unparseable date becomes an explicit missing
  value rather than the pipeline silently guessing a date and being wrong.
- **Hard `assert`s in the Verification Gate rather than just print statements** — a
  print statement can be ignored; a failed assertion stops execution and forces attention.

---

## Limitations

- Deduplication currently keeps the *first* occurrence of a duplicate `OrderID`, not
  necessarily the *most complete or most recent* one. On a dataset with real duplicates
  that differ in completeness, a more sophisticated "keep the row with fewest nulls" rule
  might be preferable.
- Outlier detection (e.g. a `Quantity` of 99,999) is not currently implemented — the pipeline
  checks *consistency* (does `TotalPrice` match the formula) but not statistical
  *plausibility* (is this value realistic at all).
- Text standardization uses Title Case as a single global rule; a dataset with legitimate
  mixed-case values (e.g. brand names like "iPhone") would need a custom exception list.
- The pipeline is designed for datasets that fit comfortably in memory via pandas; very
  large files (multi-million rows) would benefit from a chunked or Dask-based approach.

---

## Possible Future Improvements

- Add outlier/statistical-plausibility checks (e.g. flag `Quantity` or `UnitPrice` values
  beyond N standard deviations from the mean)
- Make the deduplication rule configurable (first vs. most-complete vs. most-recent)
- Add a command-line interface (`python clean.py --input file.xlsx --output clean.xlsx`)
  so the pipeline can run outside of Jupyter
- Add automated unit tests (e.g. `pytest`) for each cleaning function
- Support CSV input/output in addition to Excel

---

## License

No license has been set yet. Add one (e.g. MIT) in a `LICENSE` file if you intend for others
to reuse or contribute to this code.

---

## Credits

Built as Project 1 of the DecodeLabs Industrial Training Kit, Batch 2026.
