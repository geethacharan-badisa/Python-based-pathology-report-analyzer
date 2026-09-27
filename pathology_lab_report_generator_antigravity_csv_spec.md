# Pathology Lab Report Generator — Antigravity Implementation Specification

## 1. Project Goal

Build a small full-stack pathology report generator that:

1. Stores patient information.
2. Accepts analyser CSV files.
3. Validates and parses analyser results.
4. Maps analyser tests to internal analytes.
5. Normalizes supported units.
6. Selects the configured reference range using patient demographics.
7. Classifies results.
8. Displays results in a web dashboard.
9. Generates a PDF report.

This is an MVP/educational application.

**Do not implement disease diagnosis, treatment recommendations, or medical decision-making.**

---

# 2. Technology Stack

## Backend

Use:

- Python 3.12+
- FastAPI
- Pydantic v2
- ReportLab
- pytest
- httpx

Use Python's built-in `csv` module.

Do not add pandas unless the actual input format requires it.

## Storage

**Do not use PostgreSQL, SQLite, MongoDB, Redis, Docker, or any other database/container system.**

Use a simple file-based CSV storage layer.

Store application data as separate CSV files.

Recommended:

```text
data/
├── patients.csv
├── analytes.csv
├── analyte_mappings.csv
├── reference_ranges.csv
├── unit_conversions.csv
├── lab_results.csv
├── import_batches.csv
├── import_errors.csv
└── reports.csv
```

The uploaded analyser CSV is an input file, not the application's database.

## Frontend

Use:

- React
- TypeScript
- Vite

Preserve the existing frontend stack where practical.

---

# 3. Architecture

Use a simple modular monolith.

```text
React + Vite
      |
      | REST
      v
FastAPI
      |
      +----------------------------+
      |             |              |
   Patients      Lab Engine      Reports
                   |
          +--------+--------+
          |        |        |
       Mapping Conversion Reference
                   |
                   v
             CSV Storage
                   |
          local data/*.csv
```

Processing:

```text
Patient Data + Analyser CSV
             |
             v
       CSV Validation
             |
             v
       Analyte Mapping
             |
             v
       Unit Normalization
             |
             v
    Reference Range Lookup
             |
             v
       Classification
             |
             v
       Save to CSV
          /       \
         v         v
     Dashboard     PDF
```

Keep the system local and simple.

---

# 4. Important Development Rules

## Rule 1 — Inspect first

Before changing code:

- inspect the repository;
- identify frontend structure;
- identify existing backend structure;
- identify existing config files;
- identify reusable components;
- check whether data files already exist.

Do not rewrite working code unnecessarily.

## Rule 2 — Work incrementally

Implement in this order:

```text
1. Repository inspection
2. FastAPI foundation
3. CSV storage layer
4. Domain models
5. Reference-range logic
6. Unit conversion
7. Classification
8. Analyser CSV import
9. API
10. Tests
11. Frontend
12. PDF
13. End-to-end verification
14. Documentation
```

After each meaningful stage:

- run tests;
- fix failures;
- continue.

## Rule 3 — Keep the code small

Avoid:

- unnecessary interfaces;
- unnecessary design patterns;
- microservices;
- background queues;
- message brokers;
- external databases;
- unnecessary dependencies.

This is deliberately a small application.

## Rule 4 — Do not invent requirements

Use the simplest implementation that satisfies this specification.

Do not add unrelated features.

## Rule 5 — Do not invent clinical data

Reference ranges are configuration.

If a range does not exist:

```text
NO_REFERENCE_RANGE
```

Do not invent a medical range.

Demo values must be marked:

```text
DEMO / TEST CONFIGURATION
```

---

# 5. CSV Storage Design

The CSV files act as a lightweight persistence layer.

Every CSV file must:

- have a stable header;
- use UTF-8;
- use one record per row;
- have a unique `id` column where applicable;
- use ISO-8601 timestamps;
- use consistent date formats;
- be written atomically.

## Atomic writes

Do not directly rewrite the live CSV.

Use:

```text
read existing file
    ↓
modify in memory
    ↓
write temporary file
    ↓
replace original
```

This reduces corruption risk from interrupted writes.

---

# 6. CSV Storage Service

Create one central component:

```python
CsvStorage
```

Operations:

```python
read_all(filename)
find_by_id(filename, record_id)
append(filename, record)
update(filename, record_id, record)
delete(filename, record_id)
replace_all(filename, records)
```

Do not let every feature manually open CSV files.

All application data access should go through this component.

---

# 7. File Schemas

## patients.csv

```text
id
patient_code
full_name
date_of_birth
sex
created_at
updated_at
```

## analytes.csv

```text
id
code
name
canonical_unit
active
```

## analyte_mappings.csv

```text
id
source_system
source_test_code
source_test_name
analyte_id
active
```

## reference_ranges.csv

```text
id
analyte_id
sex
min_age
max_age
age_unit
lower_bound
upper_bound
unit
critical_low
critical_high
source_name
source_version
effective_from
effective_to
active
```

## unit_conversions.csv

```text
id
analyte_code
from_unit
to_unit
factor
offset
active
```

For the initial implementation:

```text
normalized_value = value * factor + offset
```

Add more complex conversions only when actually required.

## lab_results.csv

```text
id
patient_id
analyte_id
original_test_name
original_test_code
original_value
original_unit
normalized_value
normalized_unit
reference_range_id
classification
observation_time
import_batch_id
created_at
```

## import_batches.csv

```text
id
filename
status
total_rows
successful_rows
failed_rows
created_at
```

## import_errors.csv

```text
id
import_batch_id
row_number
field
error_code
message
```

## reports.csv

```text
id
patient_id
created_at
filename
```

Store generated PDFs under:

```text
reports/
```

Store only report metadata in `reports.csv`.

---

# 8. In-Memory Indexing

CSV is fine for this small application, but avoid scanning every file repeatedly.

For frequently used data:

```text
CSV
 ↓
Python objects
 ↓
in-memory indexes
```

Useful indexes:

```text
patient id
patient code
analyte id
analyte code
mapping source test code
reference analyte
result patient id
```

For MVP:

- load small configuration files at startup;
- rebuild or refresh indexes after writes;
- do not build a complex cache.

---

# 9. Concurrency

This application targets:

```text
single-user
or
very-low-concurrency
```

local/demo use.

Do not treat CSV storage as a multi-user production database.

If concurrent writes become necessary, a simple file lock can be added later.

Do not build distributed locking.

---

# 10. Domain Model

Use Python/Pydantic models for internal data representation.

Core entities:

```text
Patient
Analyte
AnalyteMapping
ReferenceRange
LabResult
ImportBatch
ImportError
Report
```

---

# 11. Result Classification

Allowed values:

```text
NORMAL
LOW
HIGH
CRITICAL_LOW
CRITICAL_HIGH
NO_REFERENCE_RANGE
INVALID
```

Algorithm:

```python
if value is invalid:
    INVALID

elif reference_range is missing:
    NO_REFERENCE_RANGE

elif critical_low exists and value < critical_low:
    CRITICAL_LOW

elif critical_high exists and value > critical_high:
    CRITICAL_HIGH

elif value < lower_bound:
    LOW

elif value > upper_bound:
    HIGH

else:
    NORMAL
```

Normal:

```text
lower_bound <= value <= upper_bound
```

No diagnostic conclusions.

---

# 12. Age Handling

If date of birth exists:

```text
age = age at observation_time
```

Do not use today's date for historical observations.

Reject:

- future DOB;
- invalid dates;
- negative ages.

Support:

```text
DAYS
MONTHS
YEARS
```

only if required by the configured ranges.

---

# 13. Unit Normalization

Create:

```python
UnitConversionService
```

Example:

```python
convert(
    analyte,
    value,
    from_unit,
    to_unit
)
```

Requirements:

- preserve original value;
- preserve original unit;
- store normalized value;
- store normalized unit;
- convert only explicitly supported combinations;
- reject unsupported conversions;
- never guess;
- never double-convert.

---

# 14. Reference Range Selection

Create:

```python
ReferenceRangeService
```

Input:

```text
analyte
sex
age
unit
observation_time
```

Selection:

```text
1. analyte
2. active date
3. age
4. sex
5. unit
```

Outcomes:

```text
one match
    → use range

zero matches
    → NO_REFERENCE_RANGE

multiple matches without explicit precedence
    → AMBIGUOUS_REFERENCE_RANGE
```

Never arbitrarily select the first matching row.

---

# 15. CSV Analyser Import

Logical fields:

```text
patientCode
testCode OR testName
value
unit
observationTime
```

Example:

```csv
patientCode,testCode,value,unit,observationTime
P001,GLUCOSE,102,mg/dL,2026-09-26T09:30:00
P001,HEMOGLOBIN,13.8,g/dL,2026-09-26T09:30:00
```

Actual analyser column names may differ.

Implement a small mapping layer:

```text
analyser CSV
     ↓
logical fields
     ↓
internal result
```

Unknown/ambiguous test mappings must produce errors.

---

# 16. Import Pipeline

```text
Upload
  ↓
Validate file
  ↓
Parse CSV
  ↓
Validate headers
  ↓
Validate rows
  ↓
Map analyte
  ↓
Normalize unit
  ↓
Find reference range
  ↓
Classify
  ↓
Write lab_results.csv
  ↓
Write import summary/errors
```

Use tolerant import:

```text
valid rows → saved
invalid rows → import_errors.csv
```

Never silently discard rows.

---

# 17. FastAPI Structure

```text
backend/
├── app/
│   ├── main.py
│   ├── core/
│   │   └── config.py
│   │
│   ├── storage/
│   │   └── csv_storage.py
│   │
│   ├── patients/
│   │   ├── schema.py
│   │   ├── service.py
│   │   └── router.py
│   │
│   ├── analytes/
│   ├── references/
│   ├── conversions/
│   ├── results/
│   ├── imports/
│   └── reports/
│
├── data/
├── examples/
├── reports/
├── tests/
├── requirements.txt
└── README.md
```

Do not create SQLAlchemy/database code.

---

# 18. REST API

## Patients

```http
POST /api/v1/patients
GET /api/v1/patients
GET /api/v1/patients/{id}
```

## Imports

```http
POST /api/v1/imports
GET /api/v1/imports/{id}
GET /api/v1/imports/{id}/errors
```

CSV upload:

```text
multipart/form-data
```

## Results

```http
GET /api/v1/patients/{id}/results
GET /api/v1/results/{id}
```

## Reports

```http
POST /api/v1/patients/{id}/reports
GET /api/v1/reports/{id}
```

Use Pydantic request/response models.

---

# 19. Frontend

Required pages:

```text
Dashboard
Patients
Patient Details
CSV Import
Report
```

Patient details:

```text
patient name
patient ID
age
sex
observation/report date
```

Result table:

```text
Test | Result | Unit | Reference Range | Status
```

Expanded result:

```text
Original value
Original unit
Normalized value
Normalized unit
Reference range
Classification
Source/version
Observation time
```

---

# 20. Status UI

Do not depend only on color.

Use:

```text
✓ NORMAL
↑ HIGH
↓ LOW
⚠ CRITICAL HIGH
⚠ CRITICAL LOW
? NO REFERENCE RANGE
× INVALID
```

Critical results should be prominent.

Do not display disease diagnoses.

---

# 21. PDF

Use ReportLab.

Include:

```text
PATHOLOGY LABORATORY REPORT

Patient information

Results table

Original values

Normalized values

Reference ranges

Classification

Reference source/version

Report date

Disclaimer
```

PDF files:

```text
reports/
```

Metadata:

```text
data/reports.csv
```

---

# 22. Validation

Patient:

```text
patient_code required
full_name required
date_of_birth valid
sex valid
```

CSV:

```text
required fields
numeric value
finite value
valid unit
known analyte mapping
valid observation timestamp
```

Reference range:

```text
lower <= upper
valid age range
valid dates
logical critical thresholds
```

---

# 23. Errors

Use:

```text
INVALID_CSV
INVALID_HEADER
INVALID_VALUE
INVALID_UNIT
UNSUPPORTED_UNIT
ANALYTE_NOT_FOUND
AMBIGUOUS_ANALYTE_MAPPING
REFERENCE_RANGE_NOT_FOUND
AMBIGUOUS_REFERENCE_RANGE
INVALID_REFERENCE_RANGE
PATIENT_NOT_FOUND
REPORT_GENERATION_FAILED
```

Example:

```json
{
  "code": "UNSUPPORTED_UNIT",
  "message": "Unsupported unit for analyte.",
  "field": "unit",
  "row": 8
}
```

Never expose stack traces.

---

# 24. Security / Privacy

For this local MVP:

- do not unnecessarily log patient results;
- validate upload size/type;
- prevent path traversal;
- keep application data under the project directory;
- do not expose internal file paths;
- do not commit real patient data.

This is not a production clinical deployment.

---

# 25. Demo Data

Create:

```text
examples/
├── patients.csv
├── analyser-results.csv
├── analytes.csv
├── analyte-mappings.csv
├── reference-ranges.csv
└── unit-conversions.csv
```

Use a very small dataset.

Mark all medical/reference values:

```text
DEMO / TEST CONFIGURATION
```

---

# 26. Testing

Use pytest.

Test:

### Classification

```text
normal
boundary
low
high
critical low
critical high
invalid
missing reference
```

### Conversion

```text
supported
same unit
decimal
unsupported
invalid input
```

### Reference lookup

```text
age match
sex match
unit match
date match
no match
ambiguous
```

### CSV storage

```text
read
append
update
replace
missing file
malformed CSV
atomic write
```

### Import

```text
valid CSV
invalid value
unknown test
unsupported unit
mixed valid/invalid rows
```

### API

```text
create patient
upload CSV
retrieve results
generate PDF
```

---

# 27. Local Execution

No Docker.

Backend:

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Frontend:

```bash
cd frontend
npm install
npm run dev
```

Document the Windows virtual-environment activation command in README if needed.

---

# 28. Demo Reset

Provide:

```text
scripts/reset_demo_data.py
```

It should restore:

```text
data/*.csv
```

to the clean demo state.

Never delete files outside the application's data directory.

---

# 29. Implementation Sequence

Implement exactly:

```text
1. Inspect repository
2. FastAPI setup
3. CSV storage layer
4. Domain models
5. Reference service
6. Unit conversion
7. Classification
8. CSV import
9. REST API
10. Tests
11. Frontend
12. PDF
13. End-to-end test
14. README
```

Do not begin with the frontend.

---

# 30. Working Style

For every step:

```text
1. Inspect only relevant files.
2. Make the smallest required change.
3. Run relevant tests/build.
4. Fix failures.
5. Continue.
```

Do not repeatedly reread the whole repository.

Do not modify unrelated files.

Do not perform broad refactors.

---

# 31. Final Acceptance Test

The complete workflow must work:

```text
Create Patient
      ↓
Upload Analyser CSV
      ↓
Validate CSV
      ↓
Map Test → Analyte
      ↓
Normalize Unit
      ↓
Find Reference Range
      ↓
Classify
      ↓
Write CSV data
      ↓
Dashboard
      ↓
Generate PDF
```

Checklist:

- [ ] FastAPI starts.
- [ ] CSV storage works.
- [ ] Patient creation works.
- [ ] Analyser CSV upload works.
- [ ] Errors are stored/reported.
- [ ] Analyte mapping works.
- [ ] Unit normalization works.
- [ ] Reference lookup works.
- [ ] Classification works.
- [ ] Results persist across application restarts.
- [ ] Dashboard displays results.
- [ ] PDF generation works.
- [ ] Tests pass.
- [ ] Demo data works.
- [ ] README explains setup and reset.

---

# 32. Final Constraint

This is intentionally a **small local MVP**.

Use:

```text
FastAPI
+
React
+
CSV files
+
ReportLab
```

Do not add:

```text
Docker
PostgreSQL
SQLite
MongoDB
Redis
Kafka
Kubernetes
microservices
```

unless a future requirement explicitly demands them.

Prefer:

```text
simple > clever
explicit > automatic
deterministic > AI
small > over-engineered
traceable > convenient
```
