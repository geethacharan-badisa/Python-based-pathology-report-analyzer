# Project Context & Design Rationale: Pathology Lab Report Generator

## 1. Executive Summary & Purpose

The **Pathology Lab Report Generator** is an educational, full-stack laboratory information sub-system designed to bridge the gap between raw automated clinical analyser data and formatted, human-verifiable pathology laboratory reports.

In clinical chemistry and hematology laboratories, automated analysers (such as Beckman Coulter, Roche Cobas, or Sysmex) produce raw CSV/text result files. These files often use instrument-specific shorthand codes (`GLUC`, `HGB`, `CREAT`), varying units of measurement (`mmol/L` vs. `mg/dL`, `g/L` vs. `g/dL`), and lack unified demographic reference interval evaluations.

This project implements a self-contained, auditable pipeline that:
1. Maintains patient demographic records.
2. Ingests raw analyser CSV files through a tolerant ingestion engine.
3. Maps proprietary instrument codes to canonical medical analytes.
4. Mathematically normalizes units of measure.
5. Selects appropriate reference intervals matching patient age, sex, and observation date.
6. Classifies values into deterministic clinical categories (e.g., Normal, High, Low, Critical).
7. Renders real-time dashboards and compiles official ReportLab PDF reports.

---

## 2. Fundamental Constraints & Clinical Boundary

### Explicit Medical Boundaries
* **Strict Non-Diagnostic Principle:** The application **never** performs automated medical diagnosis, suggests clinical etiologies, or recommends therapeutic treatments.
* **Deterministic Logic:** Result classifications are pure mathematical evaluations against configured boundaries (`lower_bound <= value <= upper_bound`).
* **Configured Data Attribution:** All reference intervals are configuration artifacts marked with `DEMO / TEST CONFIGURATION v1.0`. Missing reference ranges yield `NO_REFERENCE_RANGE` rather than fabricated clinical figures.

### Architectural Constraints
* **Zero External Databases:** The project strictly avoids PostgreSQL, MySQL, SQLite, MongoDB, Redis, and container orchestration (Docker/Kubernetes).
* **Pure Local CSV Persistence:** Persistence is implemented via a lightweight, file-based CSV storage engine.
* **Minimalist Tech Stack:** Built with Python 3.12+ (FastAPI), React 19 (TypeScript + Vite), Vanilla CSS design tokens, and ReportLab for PDF rendering.

---

## 3. Storage Architecture: Atomic CSV Persistence

Because traditional RDBMS engines are excluded, the system implements a reliable file-based persistence layer (`CsvStorage`) based on the following core principles:

### Atomic File Operations
Directly modifying live CSV files risks file corruption if an application crash or process interruption occurs mid-write. The storage layer uses the **temp-and-rename** pattern:
```text
1. Acquire in-process reentrant lock (threading.RLock)
2. Read existing CSV records into memory
3. Apply mutations (append, update, delete)
4. Write entire updated dataset to a temporary file in the same directory (.tmp)
5. Execute atomic replace (os.replace) onto the target CSV
6. Release lock
```
On Windows (NTFS), `os.replace` guarantees atomic filesystem metadata updating within the same directory volume.

### Schema Integrity
Nine distinct CSV tables are maintained with fixed column headers:
1. `patients.csv` (Demographics: MRN, name, DOB, sex)
2. `analytes.csv` (Canonical analytes: code, name, canonical unit)
3. `analyte_mappings.csv` (Instrument code to analyte ID translation)
4. `reference_ranges.csv` (Demographic intervals, age units, critical cutoffs)
5. `unit_conversions.csv` (Multiplicative factors and additive offsets)
6. `lab_results.csv` (Processed specimen observations)
7. `import_batches.csv` (Audit logs of CSV file ingestion runs)
8. `import_errors.csv` (Row-level failure logs)
9. `reports.csv` (PDF generation metadata)

### In-Memory Indexing
To avoid redundant disk I/O on every REST request, services maintain synchronized in-memory dictionaries:
* Patient lookup by UUID and MRN / Patient Code
* Analyte lookup by ID and canonical code
* Analyte mapping lookup by instrument code and test name
* Active reference ranges by analyte ID

---

## 4. Clinical Domain Logic & Normalization

### Demographic Age Evaluation
* Reference intervals for biomarkers vary drastically with age.
* **Age at Observation Time:** Patient age is calculated as `observation_time - date_of_birth`. Historical test runs never use today's date, preventing erroneous reference interval matching for retrospective data.
* Negative ages and future birth dates are rejected at validation.

### Unit Normalization Mechanics
Specimen results from different analysers frequently use distinct units:
$$\text{normalized\_value} = (\text{raw\_value} \times \text{factor}) + \text{offset}$$
* Conversions are unidirectional and explicit.
* Case-insensitive identical units (e.g., `mg/dL` vs `MG/DL`) normalize with factor $1.0$.
* Unsupported unit combinations are flagged as `UNSUPPORTED_UNIT` and logged to errors rather than guessed.

### Multi-Tier Reference Range Selection
Matching an observation to its reference range requires five criteria:
1. **Analyte Identity:** Target `analyte_id` must match.
2. **Date Validity:** Observation date must fall within `[effective_from, effective_to]`.
3. **Age Appropriateness:** Patient age at observation must satisfy `min_age <= age <= max_age` in the range's age unit (`YEARS`, `MONTHS`, or `DAYS`).
4. **Sex Specificity:** Range must match patient sex (`M`, `F`), or be universal (`ALL`).
5. **Unit Conformance:** Range unit must match the normalized measurement unit.

If multiple ambiguous non-precedence matches occur, the engine flags `AMBIGUOUS_REFERENCE_RANGE` to prevent arbitrary clinical range selection.

### Result Classification Hierarchy
Each result is deterministically classified:
```text
Value is null or non-finite      --> INVALID
No reference range matched        --> NO_REFERENCE_RANGE
Value < critical_low              --> CRITICAL_LOW
Value > critical_high             --> CRITICAL_HIGH
Value < lower_bound               --> LOW
Value > upper_bound               --> HIGH
lower_bound <= Value <= upper_bound --> NORMAL
```

---

## 5. Tolerant CSV Ingestion Engine

Real-world laboratory instrument feeds frequently contain formatting anomalies or malformed rows. The ingestion engine applies **tolerant batch processing**:
* **Header Aliasing:** Maps flexible header variations (`patientCode`, `patient_id`, `mrn`, `testCode`, `uom`, etc.).
* **Row-Level Fault Isolation:** A malformed row (e.g., invalid float, unknown test code) does not terminate the batch.
* **Audit Trail:**
  * Successful rows are appended to `lab_results.csv`.
  * Faulty rows are recorded in `import_errors.csv` with row number, field name, error code, and explanation.
  * Batch status is recorded as `COMPLETED`, `PARTIAL`, or `FAILED`.

---

## 6. Official PDF Report Generation

PDF reports are compiled server-side using **ReportLab** (`reportlab.platypus`):
* Standard letter dimensions with controlled clinical print margins.
* Complete patient demographic header with calculated age and MRN.
* Observation table displaying both original and normalized readings.
* Prominent visual indicators for abnormal and critical findings.
* Formal clinical disclaimer and Technologist / QA sign-off signature blocks.
* PDF files are stored in `reports/` with metadata cataloged in `data/reports.csv`.

---

## 7. User Interface Philosophy

The frontend is constructed using modern web design principles:
* **Glassmorphism & Sleek Dark Mode:** Deep clinical slate (`#0a0f1d`, `#111827`) with cyan and cobalt accents.
* **Accessible Status Badges:** Adheres to accessibility requirements by combining distinct symbols (`✓`, `↑`, `↓`, `⚠`, `?`, `×`) with color indicators.
* **Instant Demo Integration:** Single-click "Load Demo Analyser Data" allows immediate inspection of normal, abnormal, critical, and unit-converted cases without manual file preparation.
