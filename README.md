# 🔬 Pathology Lab Report Generator

<div align="center">

![Python 3.12+](https://img.shields.io/badge/Python-3.12%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115%2B-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![React 19](https://img.shields.io/badge/React-19.2-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.0%2B-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-8.3-646CFF?style=for-the-badge&logo=vite&logoColor=white)
![ReportLab](https://img.shields.io/badge/ReportLab-5.0-C71585?style=for-the-badge)
![Tests: 37 Passed](https://img.shields.io/badge/Tests-37%20Passed-10B981?style=for-the-badge)

<p align="center">
  <strong>A resilient, full-stack clinical pathology reporting engine with atomic CSV persistence, tolerant instrument ingestion, dynamic reference interval evaluations, and publication-grade PDF reporting.</strong>
</p>

</div>

---

> ⚠️ **Educational & Demonstration Notice:**  
> All reference ranges, analyte mappings, and specimen thresholds represent test configurations (`DEMO / TEST CONFIGURATION v1.0`). This system does **not** perform automated clinical disease diagnosis, treatment recommendations, or medical decision-making.

---

## 📊 Interactive System Architecture

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      REACT 19 + TYPESCRIPT (VITE)                      │
 │   Clinical Dashboard  │  Patient Records  │  CSV Ingestion  │  PDFs    │
 └───────────────────────────────────┬────────────────────────────────────┘
                                     │ REST API (JSON & Multipart)
 ┌───────────────────────────────────▼────────────────────────────────────┐
 │                            FASTAPI MONOLITH                            │
 │                                                                        │
 │   ┌───────────────────────┐               ┌────────────────────────┐   │
 │   │  Tolerant CSV Engine  │               │ Reference Range Engine │   │
 │   │  • Header Disambig    │               │ • Age at Observation   │   │
 │   │  • Row Fault-Isolation│               │ • Sex Matching         │   │
 │   │  • Analyte Translation│               │ • Boundary Checks      │   │
 │   └───────────┬───────────┘               └───────────┬────────────┘   │
 │               │                                       │                │
 │   ┌───────────▼───────────┐               ┌───────────▼────────────┐   │
 │   │ Unit Conversion Svc   │               │ Result Classification  │   │
 │   │ • Mathematical Matrix │               │ • NORMAL   │ HIGH/LOW  │   │
 │   │ • Normalization       │               │ • CRITICAL │ NO_RANGE  │   │
 │   └───────────┬───────────┘               └───────────┬────────────┘   │
 │               │                                       │                │
 │               └───────────────────┬───────────────────┘                │
 │                                   │                                    │
 │                       ┌───────────▼───────────┐                        │
 │                       │ ReportLab PDF Builder │                        │
 │                       │ • Clinical Layout     │                        │
 │                       │ • QA Signatures       │                        │
 │                       └───────────┬───────────┘                        │
 │                                   │                                    │
 │                       ┌───────────▼───────────┐                        │
 │                       │   CsvStorage Engine   │                        │
 │                       │  (Atomic os.replace)  │                        │
 │                       └───────────┬───────────┘                        │
 └───────────────────────────────────┼────────────────────────────────────┘
                                     │ Atomic Write (.tmp -> Live)
 ┌───────────────────────────────────▼────────────────────────────────────┐
 │                      LOCAL PERSISTENCE DATA DIR                        │
 │                                                                        │
 │   📄 patients.csv           📄 analytes.csv        📄 mappings.csv     │
 │   📄 reference_ranges.csv   📄 conversions.csv     📄 lab_results.csv  │
 │   📄 import_batches.csv     📄 import_errors.csv   📄 reports.csv      │
 │                                                                        │
 │   📂 reports/*.pdf (Publication-Ready Laboratory Documents)           │
 └────────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Key Functional Capabilities

* **Demographic Age at Observation Time:** Patient age is calculated specifically relative to observation timestamps, preventing erroneous demographic matching for historical specimen runs.
* **Analyte Mapping & Disambiguation:** Translates diverse analyzer codes (`GLUC`, `HGB`, `CREAT`, `POTASSIUM`) to canonical entities.
* **Linear Unit Normalization:** Applies explicit mathematical transformations:
  $$\text{Normalized Value} = (\text{Raw Value} \times \text{Factor}) + \text{Offset}$$
* **Deterministic Status Indicators:**
  * <kbd style="background:#064e3b;color:#34d399">✓ NORMAL</kbd> Values within calibrated demographic thresholds.
  * <kbd style="background:#78350f;color:#fbbf24">↑ HIGH</kbd> Measurements exceeding standard upper bounds.
  * <kbd style="background:#0c4a6e;color:#38bdf8">↓ LOW</kbd> Measurements below standard lower bounds.
  * <kbd style="background:#7f1d1d;color:#f87171">⚠ CRITICAL HIGH / LOW</kbd> Immediate clinical-alert cutoffs.
  * <kbd style="background:#1e293b;color:#94a3b8">? NO REFERENCE RANGE</kbd> Missing demographic interval.
  * <kbd style="background:#1e293b;color:#94a3b8">× INVALID</kbd> Non-numeric or non-finite measurements.
* **Tolerant Ingestion:** Valid rows are saved directly to `lab_results.csv`; malformed rows are isolated and logged to `import_errors.csv`.
* **Zero External Databases:** Completely self-contained atomic CSV file architecture. No PostgreSQL, Docker, or external cloud configurations required.

---

## ⚡ Quick Start

### 1. Launch Backend (FastAPI)

```powershell
# Navigate to backend
cd backend

# Create & activate virtual environment (Windows PowerShell)
python -m venv .venv
.venv\Scripts\Activate.ps1

# Install requirements
pip install -r requirements.txt

# Start backend server
uvicorn app.main:app --port 8000 --reload
```

* **Swagger Interactive Docs:** [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)
* **Backend Health Check:** [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health)

---

### 2. Launch Frontend (React + Vite)

```powershell
# In a separate terminal, navigate to frontend
cd frontend

# Install Node dependencies
npm install

# Start Vite dev server
npm run dev
```

* **Clinical Dashboard:** [http://127.0.0.1:5173](http://127.0.0.1:5173)

---

### 3. Reset to Clean Demo State

To clear all transactional specimen runs and reset patients/analytes to the pristine demo seed:

```powershell
# From project root
python scripts/reset_demo_data.py
```

---

## 📝 Demo Analyser CSV File

A ready-to-submit sample file is available at [`demo_analyser_input.csv`](file:///c:/skill%20palaver!/demo_analyser_input.csv):

```csv
patientCode,testCode,value,unit,observationTime
P001,GLUCOSE,102,mg/dL,2026-09-26T09:30:00
P001,HEMOGLOBIN,13.8,g/dL,2026-09-26T09:30:00
P001,CREATININE,0.9,mg/dL,2026-09-26T09:30:00
P001,POTASSIUM,4.2,mmol/L,2026-09-26T09:30:00
P001,WBC,7.2,10^3/uL,2026-09-26T09:30:00
P002,GLUCOSE,50,mg/dL,2026-09-26T10:00:00
P002,HEMOGLOBIN,11.2,g/dL,2026-09-26T10:00:00
P002,POTASSIUM,6.5,mmol/L,2026-09-26T10:00:00
P003,GLUC,5.2,mmol/L,2026-09-26T10:15:00
P003,HGB,145,g/L,2026-09-26T10:15:00
P003,CHOL,215,mg/dL,2026-09-26T10:15:00
```

### What This Tests
* **P001 (Adult Male):** Glucose (`102 mg/dL` &rarr; <span style="color:#fbbf24">**HIGH**</span>), Hemoglobin (`13.8 g/dL` &rarr; <span style="color:#34d399">**NORMAL**</span>), Creatinine (`0.9 mg/dL` &rarr; <span style="color:#34d399">**NORMAL**</span>).
* **P002 (Adult Female):** Glucose (`50 mg/dL` &rarr; <span style="color:#f87171">**CRITICAL LOW**</span>), Hemoglobin (`11.2 g/dL` &rarr; <span style="color:#38bdf8">**LOW**</span>), Potassium (`6.5 mmol/L` &rarr; <span style="color:#f87171">**CRITICAL HIGH**</span>).
* **P003 (Unit Conversion):** `GLUC 5.2 mmol/L` converted to `93.69 mg/dL` (<span style="color:#34d399">**NORMAL**</span>), `HGB 145 g/L` converted to `14.5 g/dL` (<span style="color:#34d399">**NORMAL**</span>), `CHOL 215 mg/dL` (<span style="color:#fbbf24">**HIGH**</span>).

---

## 🧪 Automated Test Suite

The comprehensive test suite covers all units, integration points, and live workflows:

```powershell
# Run all tests
pytest backend/tests
```

```text
collected 37 items

backend/tests/test_classification.py ........   [ 21%]
backend/tests/test_conversion.py ......         [ 37%]
backend/tests/test_reference_lookup.py .....    [ 51%]
backend/tests/test_csv_storage.py .....         [ 64%]
backend/tests/test_import_pipeline.py ........  [ 86%]
backend/tests/test_api.py ....                  [ 97%]
backend/tests/test_live_e2e.py .                [100%]

======================== 37 passed in 1.48s ========================
```

---

## 📑 Project Documentation Index

* 📘 [**PROJECT_CONTEXT.md**](file:///c:/skill%20palaver!/PROJECT_CONTEXT.md): Core medical boundaries, architectural constraints, and design rationale.
* 📗 [**SOFTWARE_DOCUMENTATION.md**](file:///c:/skill%20palaver!/SOFTWARE_DOCUMENTATION.md): Complete technical specifications, REST APIs, CSV schemas, and algorithmic workflows.
* 📙 [**PROJECT_REQUIREMENTS.md**](file:///c:/skill%20palaver!/PROJECT_REQUIREMENTS.md): Exhaustive package, library, and tool version inventory across Backend and Frontend.
