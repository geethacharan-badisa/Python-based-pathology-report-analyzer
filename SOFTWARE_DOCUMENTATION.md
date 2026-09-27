# Pathology Lab Report Generator — Complete Software Documentation

**System Version:** 1.0.0  
**Target Environment:** Local Standalone Deployment (Single User / Low Concurrency)  
**Specification Reference:** `pathology_lab_report_generator_antigravity_csv_spec.md`

---

## Table of Contents

1. [Architecture & System Design](#1-architecture--system-design)
2. [Data Storage & Persistence Architecture](#2-data-storage--persistence-architecture)
3. [Domain Services & Algorithmic Specifications](#3-domain-services--algorithmic-specifications)
4. [Tolerant CSV Ingestion Pipeline](#4-tolerant-csv-ingestion-pipeline)
5. [ReportLab PDF Generation Subsystem](#5-reportlab-pdf-generation-subsystem)
6. [REST API Specification](#6-rest-api-specification)
7. [Frontend Architecture & UI Specification](#7-frontend-architecture--ui-specification)
8. [Automated Verification & Test Suite](#8-automated-verification--test-suite)
9. [Error Codes & Troubleshooting Guide](#9-error-codes--troubleshooting-guide)

---

## 1. Architecture & System Design

### 1.1 Architectural Style
The application is structured as a **Modular Monolith** comprising:
* A **FastAPI REST Service** in Python 3.12+ handling clinical calculation, validation, persistence, and PDF compiling.
* A **Single Page Application (SPA)** in React 19, TypeScript, and Vite.
* A **File-based Persistence Engine** utilizing structured UTF-8 CSV tables with atomic file replacement.

```text
┌────────────────────────────────────────────────────────┐
│                   React 19 SPA (Vite)                  │
│   Dashboard  │  Patients  │  Import  │  Reports  │ Config │
└───────────────────────────┬────────────────────────────┘
                            │ REST (JSON / Multipart)
┌───────────────────────────▼────────────────────────────┐
│                    FastAPI Backend                     │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│ │ Patients API │ │ Imports API  │ │   Reports API    │ │
│ └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘ │
│        │                │                  │           │
│ ┌──────▼────────────────▼──────────────────▼─────────┐ │
│ │                   Domain Services                   │ │
│ │  Analyte Mapping │ Unit Conversion │ Classification │ │
│ │          Age Evaluation │ Reference Selection       │ │
│ └───────────────────────┬─────────────────────────────┘ │
│                         │                               │
│ ┌───────────────────────▼─────────────────────────────┐ │
│ │             CsvStorage (Atomic Engine)              │ │
│ └───────────────────────┬─────────────────────────────┘ │
└─────────────────────────┼───────────────────────────────┘
                          │ Atomic File System Replacement
┌─────────────────────────▼───────────────────────────────┐
│                    data/*.csv Tables                    │
│ patients.csv │ analytes.csv │ analyte_mappings.csv     │
│ reference_ranges.csv │ unit_conversions.csv             │
│ lab_results.csv │ import_batches.csv │ import_errors.csv│
│ reports.csv     │ reports/*.pdf (Generated PDFs)       │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Data Storage & Persistence Architecture

### 2.1 Atomic File Storage Layer (`CsvStorage`)
All persistence is routed through `app.storage.csv_storage.CsvStorage`. The storage class guarantees consistency and eliminates file corruption via atomic temporary-file writes:

```python
# Atomic Write Sequence
def _write_file_atomic(target_path, headers, rows):
    temp_file = NamedTemporaryFile(dir=target_path.parent, delete=False, encoding="utf-8")
    writer = csv.DictWriter(temp_file, fieldnames=headers)
    writer.writeheader()
    writer.writerows(rows)
    temp_file.close()
    os.replace(temp_file.name, target_path)
```

Thread concurrency is controlled via an in-process `threading.RLock()`.

### 2.2 Table Schemas

#### 1. `patients.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `pat_<uuid12>` |
| `patient_code` | String | Unique | Unique Medical Record Number (e.g. `P001`) |
| `full_name` | String | Required | Full patient name |
| `date_of_birth` | String | YYYY-MM-DD | Birth date (cannot be in future) |
| `sex` | String | Enum | `M`, `F`, `OTHER`, `UNKNOWN` |
| `created_at` | String | ISO-8601 | Record creation timestamp |
| `updated_at` | String | ISO-8601 | Last modification timestamp |

#### 2. `analytes.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `ana_<name>` (e.g. `ana_glu`) |
| `code` | String | Unique | Canonical analyte code (`GLUCOSE`) |
| `name` | String | Required | Display name (`Fasting Blood Glucose`) |
| `canonical_unit` | String | Required | Standard reference unit (`mg/dL`) |
| `active` | Boolean | Default True | Active catalog flag |

#### 3. `analyte_mappings.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `map_<uuid12>` |
| `source_system` | String | Default `ANALYSER` | Origin instrument model or system |
| `source_test_code`| String | Required | Analyser shorthand code (`GLUC`, `HGB`) |
| `source_test_name`| String | Optional | Analyser descriptive name |
| `analyte_id` | String | Foreign Key | References `analytes.id` |
| `active` | Boolean | Default True | Active translation rule |

#### 4. `reference_ranges.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `rr_<uuid12>` |
| `analyte_id` | String | Foreign Key | References `analytes.id` |
| `sex` | String | Enum | `M`, `F`, `ALL`, `ANY` |
| `min_age` | Float | &ge; 0 | Minimum demographic age inclusive |
| `max_age` | Float | &ge; min_age | Maximum demographic age inclusive |
| `age_unit` | String | Enum | `YEARS`, `MONTHS`, `DAYS` |
| `lower_bound` | Float | Required | Lower normal boundary |
| `upper_bound` | Float | &ge; lower | Upper normal boundary |
| `unit` | String | Required | Reference measurement unit |
| `critical_low` | Float | Optional | Critical low threshold |
| `critical_high` | Float | Optional | Critical high threshold |
| `source_name` | String | Required | Clinical standard / guideline name |
| `source_version`| String | Required | Version of guideline |
| `effective_from`| String | Optional | Effective start date YYYY-MM-DD |
| `effective_to` | String | Optional | Effective end date YYYY-MM-DD |
| `active` | Boolean | Default True | Active status |

#### 5. `unit_conversions.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `conv_<uuid12>` |
| `analyte_code` | String | Foreign Key | References `analytes.code` |
| `from_unit` | String | Required | Instrument unit (`mmol/L`) |
| `to_unit` | String | Required | Target canonical unit (`mg/dL`) |
| `factor` | Float | Required | Multiplicative conversion multiplier |
| `offset` | Float | Default 0.0 | Additive calibration offset |
| `active` | Boolean | Default True | Active conversion rule |

#### 6. `lab_results.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `res_<uuid12>` |
| `patient_id` | String | Foreign Key | References `patients.id` |
| `analyte_id` | String | Foreign Key | References `analytes.id` |
| `original_test_name` | String | | Test name from CSV |
| `original_test_code` | String | | Test code from CSV |
| `original_value` | Float | Numeric | Raw value from instrument |
| `original_unit` | String | | Raw unit from instrument |
| `normalized_value` | Float | Numeric | Converted canonical value |
| `normalized_unit` | String | | Canonical unit of analyte |
| `reference_range_id`| String | Optional | References `reference_ranges.id` |
| `classification` | String | Enum | Clinical status evaluation |
| `observation_time` | String | ISO-8601 | Specimen observation timestamp |
| `import_batch_id` | String | Foreign Key | References `import_batches.id` |
| `created_at` | String | ISO-8601 | Ingestion timestamp |

#### 7. `import_batches.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `batch_<uuid12>` |
| `filename` | String | Required | Uploaded filename |
| `status` | String | Enum | `COMPLETED`, `PARTIAL`, `FAILED` |
| `total_rows` | Integer | &ge; 0 | Total data rows evaluated |
| `successful_rows`| Integer| &ge; 0 | Valid persisted rows |
| `failed_rows` | Integer | &ge; 0 | Rejected error rows |
| `created_at` | String | ISO-8601 | Import timestamp |

#### 8. `import_errors.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `err_<uuid12>` |
| `import_batch_id`| String | Foreign Key | References `import_batches.id` |
| `row_number` | Integer | &ge; 2 | CSV file line index |
| `field` | String | | Invalid field name |
| `error_code` | String | | Standard error identifier |
| `message` | String | | Context explanation |

#### 9. `reports.csv`
| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | String | Primary Key | Format: `rep_<uuid12>` |
| `patient_id` | String | Foreign Key | References `patients.id` |
| `created_at` | String | ISO-8601 | Report compilation timestamp |
| `filename` | String | Required | Generated PDF filename |

---

## 3. Domain Services & Algorithmic Specifications

### 3.1 Age-at-Observation Calculation
Age is evaluated dynamically against `observation_time` using calendar subtraction:
```python
def calculate_age(dob, observation_date, age_unit="YEARS"):
    if dob > observation_date:
        raise ValueError("DOB cannot be after observation time")
    if age_unit == "DAYS":
        return float((observation_date - dob).days)
    elif age_unit == "MONTHS":
        months = (observation_date.year - dob.year) * 12 + (observation_date.month - dob.month)
        if observation_date.day < dob.day:
            months -= 1
        return float(max(0, months))
    else:  # YEARS
        years = observation_date.year - dob.year - ((observation_date.month, observation_date.day) < (dob.month, dob.day))
        return float(max(0, years))
```

### 3.2 Analyte Disambiguation Logic
When an analyser CSV row arrives with `test_code` or `test_name`:
1. Search active mappings in `analyte_mappings.csv` where `source_test_code.upper() == test_code.upper()` or `source_test_name.lower() == test_name.lower()`.
2. Fallback to direct code/name matching against `analytes.csv`.
3. If matched mappings point to more than one unique `analyte_id`, return error `AMBIGUOUS_ANALYTE_MAPPING`.
4. If zero matches are found, return error `ANALYTE_NOT_FOUND`.

### 3.3 Unit Normalization
* If `from_unit.lower() == to_unit.lower()`, result is `(value, to_unit)`.
* Search active rules in `unit_conversions.csv` matching `analyte_code`, `from_unit`, and `to_unit`.
* Apply:
  $$\text{normalized\_value} = (\text{value} \times \text{factor}) + \text{offset}$$
* If no rule exists, abort conversion and flag `UNSUPPORTED_UNIT`.

### 3.4 Reference Range Matching
Given an observation for an analyte:
1. Filter active reference ranges matching `analyte_id`.
2. Filter by observation timestamp within `[effective_from, effective_to]`.
3. Calculate patient age at observation and filter where `min_age <= patient_age <= max_age`.
4. Filter by patient sex (`range.sex == patient.sex` or `range.sex in ('ALL', 'ANY')`).
5. Filter by unit matching (`range.unit == normalized_unit`).
6. **Disambiguation:** If multiple matches occur, sex-specific matches take precedence over universal (`ALL`) matches. If ambiguities remain, return `AMBIGUOUS_REFERENCE_RANGE`.
7. If zero matches occur, return `NO_REFERENCE_RANGE`.

### 3.5 Result Classification Algorithm
Classification is purely deterministic:
```python
if value is None or not math.isfinite(value):
    return "INVALID"
elif reference_range is None:
    return "NO_REFERENCE_RANGE"
elif reference_range.critical_low is not None and value < reference_range.critical_low:
    return "CRITICAL_LOW"
elif reference_range.critical_high is not None and value > reference_range.critical_high:
    return "CRITICAL_HIGH"
elif value < reference_range.lower_bound:
    return "LOW"
elif value > reference_range.upper_bound:
    return "HIGH"
else:
    return "NORMAL"
```

---

## 4. Tolerant CSV Ingestion Pipeline

The import pipeline processes analyser batches using a robust tolerance architecture:

```text
Upload CSV
    │
    ▼
Header Normalization (alias resolution for patientCode, testCode, value, unit, observationTime)
    │
    ▼
Row-by-Row Evaluation Loop
    ├── 1. Validate required fields present
    ├── 2. Validate numeric finite value
    ├── 3. Parse and validate observation timestamp
    ├── 4. Lookup patient by MRN / ID
    ├── 5. Verify observation date >= patient DOB
    ├── 6. Disambiguate test to canonical Analyte
    ├── 7. Convert and normalize units
    ├── 8. Match demographic Reference Range
    └── 9. Classify result
    │
    ├── [Success] ──> Append record to lab_results.csv
    └── [Failure] ──> Append row failure to import_errors.csv
    │
    ▼
Update import_batches.csv (status = COMPLETED / PARTIAL / FAILED)
```

---

## 5. ReportLab PDF Generation Subsystem

The PDF generator (`app.reports.pdf_generator`) compiles pathology laboratory reports to letter-sized documents:
* **Margins:** 0.5 inch (36pt) margins maximizing clinical readability.
* **Palette:** Clinical Navy (`#0f172a`), Cyan accent (`#0284c7`), Slate borders (`#cbd5e1`).
* **Header:** Laboratory identity, Report ID, Generation Date, Record Status.
* **Patient Demographics:** MRN, Full Name, Biological Sex, Date of Birth, Age at report, Test count.
* **Observation Table:**
  * Analyte title and code
  * Normalized result (with original measurement displayed if converted)
  * Units
  * Reference interval and source/version
  * Status badge (`✓ NORMAL`, `↑ HIGH`, `↓ LOW`, `⚠ CRITICAL`)
* **Legal Disclaimer:** Explicit warning that values represent demo configurations and are not for medical diagnosis without physician review.
* **Signatures:** technologist signature line and QA verification hashes.

---

## 6. REST API Specification

### Base URL: `/api/v1`

### 6.1 Patient Endpoints
* `GET /api/v1/patients`
  * Response: `200 OK` &rarr; Array of `PatientResponse`
* `POST /api/v1/patients`
  * Body: `PatientCreate` (`patient_code`, `full_name`, `date_of_birth`, `sex`)
  * Response: `201 Created` &rarr; `PatientResponse`
  * Error: `400 Bad Request` (Duplicate code, invalid DOB, future DOB)
* `GET /api/v1/patients/{patient_id}`
  * Response: `200 OK` &rarr; `PatientResponse`
  * Error: `404 Not Found`

### 6.2 Import Endpoints
* `POST /api/v1/imports`
  * Content-Type: `multipart/form-data` (`file: UploadFile`)
  * Response: `201 Created` &rarr; `ImportSummaryResponse`
  * Error: `400 Bad Request` (Invalid CSV structure, missing required headers)
* `GET /api/v1/imports`
  * Response: `200 OK` &rarr; Array of `ImportBatchResponse`
* `GET /api/v1/imports/{batch_id}`
  * Response: `200 OK` &rarr; `ImportBatchResponse`
* `GET /api/v1/imports/{batch_id}/errors`
  * Response: `200 OK` &rarr; Array of `ImportErrorResponse`

### 6.3 Results Endpoints
* `GET /api/v1/patients/{patient_id}/results`
  * Response: `200 OK` &rarr; Array of `EnrichedLabResultResponse`
* `GET /api/v1/results`
  * Response: `200 OK` &rarr; Array of `EnrichedLabResultResponse`
* `GET /api/v1/results/{result_id}`
  * Response: `200 OK` &rarr; `EnrichedLabResultResponse`

### 6.4 Reports Endpoints
* `POST /api/v1/patients/{patient_id}/reports`
  * Response: `201 Created` &rarr; `ReportResponse`
* `GET /api/v1/patients/{patient_id}/reports`
  * Response: `200 OK` &rarr; Array of `ReportResponse`
* `GET /api/v1/reports`
  * Response: `200 OK` &rarr; Array of `ReportResponse`
* `GET /api/v1/reports/{report_id}`
  * Response: `200 OK` &rarr; `ReportResponse`
* `GET /api/v1/reports/{report_id}/pdf`
  * Response: `200 OK` &rarr; `application/pdf` binary stream

### 6.5 Analytes & Configuration Endpoints
* `GET /api/v1/analytes` &rarr; Array of `AnalyteResponse`
* `GET /api/v1/analytes/mappings` &rarr; Array of `AnalyteMappingResponse`
* `GET /api/v1/references` &rarr; Array of `ReferenceRangeResponse`

---

## 7. Frontend Architecture & UI Specification

### 7.1 Framework & Tools
* **React 19 & TypeScript:** Strict typing with `verbatimModuleSyntax`.
* **Vite 8:** Lightning-fast HMR and optimized production bundling.
* **Lucide React:** Clinical & administrative vector iconography.
* **Vanilla CSS Design System:** Custom HSL design tokens, responsive CSS grid, glassmorphism, and accessibility-compliant status badges.

### 7.2 Core Views
1. **Dashboard (`DashboardView.tsx`):** Operational metrics (Active Patients, Total Tests, Critical Findings, Batches), quick actions, and recent feeds.
2. **Patient Directory (`PatientsView.tsx`):** Real-time search by name or MRN, demographics table, and direct report generation shortcuts.
3. **Patient Clinical Record (`PatientDetailsView.tsx`):** Demographic summary, critical value warnings, expandable observation rows showing original vs normalized measurements, and PDF history.
4. **Analyser Import Hub (`CsvImportView.tsx`):** Drag-and-drop CSV upload, "Load Demo Analyser Data" button, batch summary card, and tolerant error viewer.
5. **PDF Reports Archive (`ReportsView.tsx`):** Comprehensive list of compiled reports with in-browser viewing and download options.
6. **Configuration & Rules Viewer (`ConfigView.tsx`):** Read-only view of canonical analytes and reference ranges.

---

## 8. Automated Verification & Test Suite

The test suite contains **37 pytest automated tests** executed via:
```powershell
pytest backend/tests
```

### Test Breakdown
1. **`test_classification.py`:** Normal, boundary values, low, high, critical low, critical high, invalid numbers, missing reference range.
2. **`test_conversion.py`:** Same-unit identity, case-insensitive match, decimal factors, unsupported units, unknown analytes.
3. **`test_reference_lookup.py`:** Age boundaries, male vs female ranges, unit mismatches, future DOB rejection, unmatched analytes.
4. **`test_csv_storage.py`:** Empty reads, append, find by ID, in-place update, delete, atomic replacement.
5. **`test_import_pipeline.py`:** Valid batch ingestion, unit conversions during import, NaN/infinite value rejection, unknown analyte handling, unsupported unit rejection, mixed valid/invalid tolerant batch handling, header validation.
6. **`test_api.py`:** Patient registration, CSV upload multipart handling, observation retrieval, PDF report compiling.
7. **`test_live_e2e.py`:** Complete end-to-end user journey across running FastAPI service.

---

## 9. Error Codes & Troubleshooting Guide

| Error Code | Field | Trigger Condition | Solution |
|---|---|---|---|
| `INVALID_CSV` | `file` | Empty file or non-CSV encoding | Provide a non-empty UTF-8 encoded CSV file |
| `INVALID_HEADER` | `headers` | Missing required columns | Ensure `patientCode`, `testCode`/`testName`, `value`, `unit`, `observationTime` are present |
| `INVALID_VALUE` | `value` / `observationTime` | Non-numeric or infinite number / malformed timestamp | Provide valid decimal numbers and ISO-8601 timestamps |
| `INVALID_UNIT` | `unit` | Blank unit field | Provide unit of measurement |
| `PATIENT_NOT_FOUND` | `patientCode` | Patient code does not exist in `patients.csv` | Register patient before importing results |
| `ANALYTE_NOT_FOUND` | `testCode` | Instrument test code has no mapping | Add mapping in `analyte_mappings.csv` |
| `AMBIGUOUS_ANALYTE_MAPPING` | `testCode` | Multiple active mappings resolve to different analytes | Ensure test code mapping is unique |
| `UNSUPPORTED_UNIT` | `unit` | No unit conversion exists to canonical unit | Configure conversion factor in `unit_conversions.csv` |
| `AMBIGUOUS_REFERENCE_RANGE` | `referenceRange` | Multiple conflicting reference ranges match patient demographics | Add specific sex or age bounds to reference ranges |
| `REPORT_GENERATION_FAILED` | `report` | PDF document build error | Check file write permissions in `backend/reports/` |
