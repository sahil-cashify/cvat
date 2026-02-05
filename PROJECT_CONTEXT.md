# CVAT Project Context

This document provides a high-level overview of the **Computer Vision Annotation Tool (CVAT)** codebase so you can understand the whole project: purpose, architecture, main components, and how they connect.

---

## 1. What is CVAT?

**CVAT** is an interactive **video and image annotation tool** for computer vision. It is used to:

- Create and manage **projects** and **tasks** that hold media (images, video) and labels
- Annotate with shapes (rectangles, polygons, polylines, points, cuboids, masks, skeletons, tags)
- Run **automatic labeling** via serverless ML models (detectors, segmentors, trackers)
- Import/export annotations in many formats (COCO, YOLO, Pascal VOC, etc.)
- Support **organizations**, **quality control**, **consensus**, and **webhooks**

The codebase is a **full-stack application**: Django REST backend, React frontend, and several TypeScript libraries. It runs in Docker with PostgreSQL, Redis/KvRocks, optional ClickHouse, and Traefik as reverse proxy.

---

## 2. Repository Layout (Top Level)

| Path | Purpose |
|------|--------|
| **`cvat/`** | Django backend (Python): API, business logic, DB models, workers |
| **`cvat-ui/`** | Main React SPA: pages, Redux, UI components |
| **`cvat-core/`** | Client-side TS library: API client, sessions, annotations, frames |
| **`cvat-canvas/`** | 2D annotation canvas (drawing, shapes, SVG) |
| **`cvat-canvas3d/`** | 3D annotation (point clouds) |
| **`cvat-data/`** | Client-side dataset/frame handling (e.g. zip, chunked loading) |
| **`cvat-sdk/`** | Python SDK for scripting and automation |
| **`cvat-cli/`** | Command-line interface (pip-installable) |
| **`serverless/`** | Nuclio serverless functions (auto-annotation: detectors, segmentors, trackers) |
| **`components/`** | Analytics (Grafana, Vector, ClickHouse config) |
| **`tests/`** | Cypress E2E, Python REST API tests |
| **`helm-chart/`** | Kubernetes deployment |
| **`site/`** | Documentation (e.g. for docs.cvat.ai) |

Root **`package.json`** uses **Yarn workspaces**: `cvat-data`, `cvat-core`, `cvat-canvas`, `cvat-canvas3d`, `cvat-ui`.

---

## 3. Backend (Django) — `cvat/`

### 3.1 Entrypoint and Process Model

- **`backend_entrypoint.sh`** is the container entrypoint. It supports:
  - **`init`** — DB migrations, Redis migrations, sync periodic jobs, optional ClickHouse init
  - **`run server`** — collectstatic + Supervisord **server** (nginx + uvicorn ASGI)
  - **`run worker <queues>`** — Supervisord **workers** (RQ)

- **`supervisord/server.conf`** runs **nginx** (reverse proxy to app) and **uvicorn** (Django ASGI: `cvat.asgi:application`).
- **`supervisord/worker.conf`** runs **RQ workers** for queues such as: `notifications`, `cleaning`, `import`, `export`, `annotation`, `webhooks`, `quality_reports`, `chunks`, `consensus`.

### 3.2 Main Django Apps (`cvat/apps/`)

| App | Role |
|-----|------|
| **`engine`** | Core: Projects, Tasks, Jobs, Data, Segments, Labels, Annotations, Issues, Comments, Cloud Storages, Assets, Annotation Guides. REST ViewSets, serializers, background jobs (import/export, chunks). |
| **`dataset_manager`** | Dataset/annotation format handling: import/export, format implementations (e.g. `formats/coco.py`, `yolo.py`, `pascal_voc.py`), project/task bindings. |
| **`iam`** | Auth: login, registration, sessions, basic auth. |
| **`organizations`** | Orgs, members, invitations. |
| **`lambda_manager`** | “Lambdas” = auto-annotation functions (Nuclio). List/call/cancel, RQ jobs. |
| **`webhooks`** | Outgoing webhooks (delivery workers). |
| **`quality_control`** | Quality settings, reports, conflicts. |
| **`consensus`** | Consensus tasks and merging. |
| **`events`** | Analytics events (e.g. sent to Vector/ClickHouse). |
| **`access_tokens`** | API access tokens and policy (used by REST permission/auth). |
| **`redis_handler`** | RQ job status, request tracking. |
| **`health`** | Health check backends. |
| **`log_viewer`** | Optional log viewer integration. |

### 3.3 URL Routing (`cvat/urls.py`)

- **`/admin/`** — Django admin  
- **`/`** — engine URLs (redirect to UI, plus API under `/api/`)  
- **`/`** — redis_handler  
- **`/django-rq/`** — RQ dashboard  
- Optional: log_viewer, events, lambda_manager, webhooks, quality_control, consensus, access_tokens, health, profiler  

**Engine** (`cvat/apps/engine/urls.py`) mounts the main REST API under **`/api/`**:

- **Router**: projects, tasks, jobs, users, server, issues, comments, labels, cloudstorages, assets, guides  
- **Schema/docs**: `/api/schema/`, `/api/swagger/`, `/api/docs/`  
- **IAM** and **organizations** also under `/api/`

### 3.4 Core Data Model (Engine)

- **Project** — optional container; has Labels (with attributes, sublabels, skeletons).  
- **Task** — belongs to optional Project; has **Data** (video/images), **Segments**.  
- **Segment** — slice of task frames.  
- **Job** — one job per segment; assignee, stage (annotation/validation/acceptance), state. Ground-truth and consensus-replica job types exist.  
- **Data** — references to client/server/remote files, chunking, dimensions (2D/3D).  
- **Label**, **AttributeSpec**, **Skeleton** — label taxonomy.  
- **Shape**, **Annotation** — actual annotations (rect, polygon, etc.).  
- **Issue**, **Comment** — review/feedback.  
- **CloudStorage** — S3, GCS, Azure.  
- **Asset**, **AnnotationGuide** — binary assets and markdown guides.

Background jobs (in **`engine/background.py`**, etc.) handle: **import**, **export**, **chunks** (frame chunks), and other long-running work via **django-rq** and Redis.

### 3.5 Permissions and Auth

- **REST**: Token, session, basic auth; **access_tokens** app for API token auth and policy.  
- **OPA (Open Policy Agent)** — container `cvat_opa` loads rules from backend (`/api/auth/rules`); used for fine-grained authorization.  
- Permissions are enforced in views and via Rego policies under `*/rules/*.rego` in the relevant apps.

### 3.6 Settings

- **`cvat/settings/base.py`** — INSTALLED_APPS, REST_FRAMEWORK, DB, Redis, CORS, etc.  
- **`development.py`** / **`production.py`** / **`testing*.py`** — environment-specific.  
- **`manage.py`** uses `cvat.settings.development` by default.

---

## 4. Frontend (React SPA) — `cvat-ui/`

### 4.1 Stack

- **React 18**, **Redux** (thunk), **React Router v5**  
- **Ant Design (antd)**  
- **TypeScript**  
- **cvat-core** (API and domain logic), **cvat-canvas** (2D), **cvat-canvas3d** (3D), **cvat-data** (data loading)

### 4.2 Entry and Routing

- **`src/index.tsx`** — creates Redux store (`createCVATStore(createRootReducer)`), wraps app in `Provider` and `BrowserRouter`, renders `PluginsEntrypoint` and `ReduxAppWrapper` (main `CVATApplication`).
- **`src/components/cvat-app.tsx`** — main app component; **`<Switch>`** defines routes, e.g.:
  - **Auth**: `/auth/login`, `/auth/register`, `/auth/logout`, password reset, email confirmation  
  - **Projects**: `/projects`, `/projects/create`, `/projects/:id`, webhooks, guide, quality-control, analytics  
  - **Tasks**: `/tasks`, `/tasks/create`, `/tasks/:id`, quality-control, analytics, consensus, create job, guide  
  - **Annotation**: `/tasks/:tid/jobs/:jid` → **AnnotationPageContainer**  
  - **Jobs**: `/jobs`  
  - **Cloud storages**: `/cloudstorages`, create/update  
  - **Organization**: `/organization`, invitations, requests, webhooks  
  - **Profile**, **Models** (`/models`)

### 4.3 State and Data Flow

- **Reducers** in `src/reducers/`: e.g. `annotation-reducer`, `auth-reducer`, `tasks-reducer`, `projects-reducer`, `jobs-reducer`, `models-reducer`, `formats-reducer`, etc.  
- **Actions** in `src/actions/`: same domains (e.g. `tasks-actions`, `annotation-actions`, `models-actions`).  
- **Containers** in `src/containers/` connect components to Redux.  
- **cvat-core** is the API layer: it talks to the REST API and exposes **Project**, **Task**, **Job**, **FrameData**, **Annotations**, **lambda** (run/cancel/list), **server** (about, formats, login, etc.). The UI uses this instead of calling HTTP directly in many places.

### 4.4 Annotation Page

- **AnnotationPageContainer** → **StandardWorkspace** (or 3D workspace).  
- **cvat-canvas** is used for 2D drawing (shapes, tools, zoom, etc.).  
- **cvat-canvas3d** for point clouds.  
- Annotation state is synced with the backend (load/save annotations, frame changes).

---

## 5. Client Libraries (Workspaces)

| Package | Role |
|--------|------|
| **cvat-core** | API client (server-proxy), session/project/job abstractions, annotations, frames, labels, ML models, cloud storage, webhooks, quality, consensus, plugins. Single entry: `cvat-core/src/index.ts`. |
| **cvat-canvas** | 2D canvas: drawing, shapes (rect, polygon, etc.), tools, zoom, keyframes. Used by the annotation page. |
| **cvat-canvas3d** | 3D canvas for point cloud annotation. |
| **cvat-data** | Dataset/frame loading and handling (e.g. zip, chunked data) used by the UI. |

---

## 6. API and SDK

- **REST API**: Versioned (e.g. 2.0), documented with **drf-spectacular** (OpenAPI). Schema at **`/api/schema/`**, **`/api/swagger/`**, **`/api/docs/`**.  
- **`cvat/schema.yml`** — OpenAPI 3 YAML (generated or maintained for compatibility).  
- **Python SDK** (`cvat-sdk`): generated from OpenAPI; used for scripting and **cvat-cli**.  
- **CLI** (`cvat-cli`): pip-installable; uses SDK to create tasks, upload data, etc.

---

## 7. Serverless / Auto-Annotation

- **Nuclio** functions live under **`serverless/`** (e.g. `pytorch/`, `openvino/`, `onnx/`, `tensorflow/`).  
- Each function has **nuclio** config (`function.yaml`, `main.py`, `model_handler.py`).  
- **lambda_manager** app: registers functions, exposes “run/call/cancel” via API; **events** app handles `handle_function_call`.  
- Backend triggers these as “lambdas” for auto-labeling (detection, segmentation, tracking, etc.).

---

## 8. Docker and Runtime

- **`docker-compose.yml`** defines:
  - **cvat_db** (PostgreSQL 15)  
  - **cvat_redis_inmem** (Redis)  
  - **cvat_redis_ondisk** (KvRocks)  
  - **cvat_server** — backend container (`init run server`)  
  - **cvat_worker_*** — multiple RQ worker containers (import, export, annotation, webhooks, quality_reports, chunks, consensus, notifications, cleaning)  
  - **cvat_ui** — frontend (served e.g. on port 8000)  
  - **traefik** — reverse proxy (e.g. 8080); routes `/api/`, `/static/`, etc. to server; rest to UI  
  - **cvat_opa** — Open Policy Agent  
  - **cvat_clickhouse**, **cvat_vector**, **cvat_grafana** — analytics (events → Vector → ClickHouse → Grafana)

- **Volumes**: `cvat_db`, `cvat_data`, `cvat_keys`, `cvat_logs`, `cvat_inmem_db`, `cvat_events_db`, `cvat_cache_db`.

---

## 9. Key Flows (Summary)

1. **User opens app** → Traefik → UI (React). Login/register via IAM (session/token).  
2. **Projects/Tasks** → REST API (engine ViewSets) → PostgreSQL; list/create/update in UI via cvat-core and Redux.  
3. **Open a job** → Annotation page loads job and frames; cvat-canvas (or 3d) for drawing; annotations saved via engine API.  
4. **Import data** → Upload or cloud storage → engine + dataset_manager; **import** worker processes files and creates chunks.  
5. **Export** → **export** worker uses dataset_manager formats (COCO, YOLO, etc.) and produces downloadable file.  
6. **Auto-annotation** → User runs “lambda” from UI → lambda_manager calls Nuclio function → results written back as annotations.  
7. **Quality / Consensus** → quality_control and consensus apps and workers; UI under quality-control and consensus routes.

---

## 10. Where to Look for Specific Features

| Need | Look at |
|------|--------|
| REST endpoints for projects/tasks/jobs | `cvat/apps/engine/views.py`, `urls.py` |
| DB models (Project, Task, Job, Shape, etc.) | `cvat/apps/engine/models.py` |
| Import/export formats | `cvat/apps/dataset_manager/formats/` |
| Background jobs (import, export, chunks) | `cvat/apps/engine/background.py`, `rq.py`; `dataset_manager` |
| Auth and permissions | `cvat/apps/iam/`, `cvat/apps/access_tokens/`, OPA rules in `*/rules/*.rego` |
| Auto-annotation (lambdas) | `cvat/apps/lambda_manager/`, `serverless/` |
| UI pages and routes | `cvat-ui/src/components/cvat-app.tsx` |
| Annotation canvas (2D) | `cvat-canvas/`, usage in `cvat-ui` annotation page |
| API client and domain logic | `cvat-core/src/` (e.g. `api.ts`, `session.ts`, `annotations*.ts`) |
| Redux state and server calls | `cvat-ui/src/actions/`, `reducers/` |
| How server/workers start | `backend_entrypoint.sh`, `supervisord/server.conf`, `worker.conf` |

---

## 11. Conventions and Tech Summary

- **Backend**: Django 4.x, DRF, django-rq, PostgreSQL, Redis/KvRocks, ASGI (uvicorn).  
- **Frontend**: React 18, Redux, TypeScript, Ant Design, React Router 5.  
- **Monorepo**: Yarn workspaces for UI and core/canvas/data libs; Python backend and SDK/CLI in same repo.  
- **Deployment**: Docker Compose (and Helm for Kubernetes).  
- **API**: REST, OpenAPI 3, token/session/basic auth; optional API tokens and OPA.

Using this document you can quickly locate the part of the project that implements a given feature and see how it fits into the overall system.
