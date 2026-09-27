# Project Requirements & Dependencies Specification

This document details all external libraries, packages, runtimes, and developer dependencies utilized across both the **Backend (Python)** and **Frontend (Node.js / React)** layers of the Pathology Lab Report Generator.

---

## 1. System Runtime Requirements

| Component | Minimum Version | Verified Active Version | Description |
|---|---|---|---|
| **Python** | `3.12.0+` | `3.14.6` | Core backend runtime environment |
| **Node.js** | `18.0.0+` | `24.15.0` | Frontend build and development runtime |
| **npm** | `9.0.0+` | `11.12.1` | Node package manager |
| **Operating System** | Cross-platform | Windows 11 (64-bit) | Compatible with Windows, Linux, and macOS |

> **Note on Databases:** No external database engines (PostgreSQL, MySQL, SQLite, MongoDB, Redis) are required. The system operates on native local CSV storage.

---

## 2. Backend Dependencies (Python)

Defined in [`backend/requirements.txt`](file:///c:/skill%20palaver!/backend/requirements.txt):

```text
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
pydantic>=2.8.0
reportlab>=4.2.0
pytest>=8.3.0
httpx>=0.27.0
python-multipart>=0.0.9
```

### Detailed Package Analysis

| Package | Version Installed | License | Purpose & Role in Architecture |
|---|---|---|---|
| **`fastapi`** | `0.141.1` | MIT | High-performance asynchronous web framework providing routing, dependency injection, and automatic OpenAPI / Swagger interactive documentation. |
| **`uvicorn[standard]`** | `0.54.0` | BSD-3-Clause | Lightning-fast ASGI web server implementation powering FastAPI. The `[standard]` extra includes `httptools` and `watchfiles`. |
| **`pydantic`** | `2.13.5` | MIT | Modern data parsing and schema validation using Python type annotations. Powers domain models, input sanitation, and response serialization. |
| **`pydantic-core`** | `2.46.5` | MIT | Rust-backed acceleration engine for Pydantic v2 validation logic. |
| **`reportlab`** | `5.0.1` | BSD | Enterprise-grade PDF generation library. Used in `pdf_generator.py` to compile formatted pathology laboratory reports. |
| **`pillow`** | `12.3.0` | HPND | Image processing library required by ReportLab for graphic rendering and canvas assets. |
| **`pytest`** | `9.1.1` | MIT | Comprehensive testing framework used to execute the 37 automated test suites. |
| **`httpx`** | `0.28.1` | BSD-3-Clause | Next-generation HTTP client supporting sync and async requests. Used by FastAPI `TestClient` and live E2E test scripts. |
| **`python-multipart`** | `0.0.32` | Apache 2.0 | Multipart form-data parser required by FastAPI for streaming CSV file uploads (`UploadFile`). |

### Backend Installation Command

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

---

## 3. Frontend Dependencies (Node.js & React)

Defined in [`frontend/package.json`](file:///c:/skill%20palaver!/frontend/package.json):

### 3.1 Production Dependencies

| Package | Version Installed | License | Purpose & Role in Architecture |
|---|---|---|---|
| **`react`** | `19.2.8` | MIT | Core UI library for component-driven frontend architecture. |
| **`react-dom`** | `19.2.8` | MIT | DOM-specific rendering methods for React. |
| **`lucide-react`** | `1.48.0` | ISC | Modern, clean vector iconography for clinical laboratory interfaces (flasks, status badges, alerts, navigation icons). |

### 3.2 Development & Build Dependencies

| Package | Version Installed | License | Purpose & Role in Architecture |
|---|---|---|---|
| **`vite`** | `8.3.0` | MIT | Next-generation frontend tooling and bundler providing rapid Hot Module Replacement (HMR) and optimized build outputs. |
| **`@vitejs/plugin-react`**| `6.1.1` | MIT | Official Vite plugin for Fast Refresh and JSX transformation with React 19. |
| **`typescript`** | `6.0.2` | Apache 2.0 | Static type-checking system ensuring robust interfaces, API contract adherence, and code quality. |
| **`@types/react`** | `19.2.18` | MIT | TypeScript type definitions for React. |
| **`@types/react-dom`** | `19.2.7` | MIT | TypeScript type definitions for React DOM. |
| **`@types/node`** | `24.13.3` | MIT | TypeScript type definitions for Node.js APIs. |

### Frontend Installation Command

```powershell
cd frontend
npm install
```

---

## 4. Summary Dependency Graph

```text
Pathology Lab Report Generator
├── Backend (Python 3.12+)
│   ├── fastapi (Web framework)
│   │   ├── starlette
│   │   └── pydantic (Validation engine)
│   ├── uvicorn (ASGI server)
│   ├── reportlab (PDF generator)
│   │   └── pillow
│   ├── python-multipart (File uploads)
│   ├── pytest (Test suite runner)
│   └── httpx (HTTP client / Integration testing)
│
└── Frontend (Node.js 18+)
    ├── react 19 (UI framework)
    ├── react-dom (DOM rendering)
    ├── lucide-react (Clinical iconography)
    ├── typescript (Type safety)
    └── vite (Build tool & dev server)
```
