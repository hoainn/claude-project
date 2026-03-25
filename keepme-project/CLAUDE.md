# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

KeepMe is a multi-service SaaS platform for fitness/wellness facility management. The project consists of multiple independent repositories within this directory:

| Directory | Description | Stack |
|-----------|-------------|-------|
| `keepme-react-v4/` | Frontend (latest) | React 19.1, TypeScript 5.8, Vite 7.1, Zustand, Zod |
| `keepme-v4/` | Backend API (latest) | Laravel 12, PHP 8.3, MySQL 8, MongoDB 7 |
| `keepme/` | Legacy backend | Laravel 8, PHP 7.3/8.0 |
| `keepme-services/` | Webhook microservices | Fastify 5, TypeScript, Kafka, Vitest |
| `keepme-python-server/` | Legacy cronjobs | Python 2.7, PHP |
| `keepme-infra/` | Infrastructure as Code | Terraform, Kubernetes, Helm |
| `keepme-argo/` | ArgoCD workflows | Kubernetes manifests |
| `charts/` | Helm charts (all services) | Helm 3 |
| `antares-data-scraper/` | Airflow data extraction | Python, Apache Airflow, BeautifulSoup |
| `antares-playwrite-service/` | Browser automation API | Python, FastAPI, Playwright |
| `antares-project/` | Multi-service automation | Node.js, Python, MongoDB, React |

### Antares Sub-projects (`antares-project/`)
- `antares-automation/` — CRM workflow automation & lead management (Express, MongoDB, Redis)
- `antares-chatbot/` — Chatbot service (Docker multi-env)
- `antares-control-centre/` — MERN admin panel for client/region/location management
- `antares-llm/` — LLM/RAG service (FastAPI, LangChain, vector stores)

## Build & Run Commands

### Frontend (keepme-react-v4)
```bash
cd keepme-react-v4
npm install
npm run dev          # Dev server on port 5173
npm run build        # tsc -b && vite build
npm run lint         # ESLint
npm run preview      # Preview production build
```

### Backend API (keepme-v4)
```bash
cd keepme-v4
cp .env.example .env
docker-compose build && docker-compose up -d
docker-compose exec api composer install
docker-compose exec api php artisan migrate
# API available at http://localhost:8080
# Health check: curl http://localhost:8080/up
```

Docker services: `api` (PHP 8.3 FPM), `nginx`, `mysql`, `mongodb`, `queue`, `scheduler`

```bash
docker-compose logs queue                          # View queue logs
docker-compose logs scheduler                      # View scheduler logs
docker-compose exec api php artisan <command>      # Run artisan commands
docker-compose exec api php artisan queue:restart   # Restart queue workers
```

### Backend Testing (keepme-v4)
```bash
docker-compose exec api php artisan test                    # All tests
docker-compose exec api php artisan test --filter=TestName  # Single test
docker-compose exec api php artisan test tests/Feature/     # Feature tests only
```
Tests use SQLite `:memory:` with `RefreshDatabase`. Queue is `sync`, cache is `array`.

### Node.js Microservices (keepme-services)
```bash
cd keepme-services/src/km-webhook-service
npm install
npm run build        # Compile TypeScript
npm run start        # Run compiled code
npm run tsnode       # Run with ts-node
npm run test         # Vitest tests
npm run test:watch   # Watch mode
npm run test:coverage
npm run lint         # ESLint
npm run lint:fix     # Auto-fix
```

Services run on port 10002 with Prometheus metrics on port 9464.

### Antares Data Scraper
```bash
cd antares-data-scraper
docker-compose up -d   # Airflow webserver on http://127.0.0.1:8080 (admin/admin)
```

### Antares Playwright Service
```bash
cd antares-playwrite-service
docker-compose up -d   # FastAPI on http://localhost:8000/docs
```

## Architecture

### Dual API Pattern
The frontend communicates with two different backends:
- **Node.js API** (`api.keepme.ai/api/v1`) — Authentication only (login/logout)
- **V4 API** (`api-dev-v4.keepme.ai/api`) — All other endpoints

Environment variables: `VITE_API_BASE_URL` (V4), `VITE_NODE_API_BASE_URL` (Node)

Centralized config in `keepme-react-v4/src/config/api.config.ts`.

### Frontend Architecture (keepme-react-v4)

**Routing:** React Router v7 with `createBrowserRouter`. All routes except `/login` are wrapped in `ProtectedRoute` (checks `authService.isAuthenticated()` via localStorage).

**API Layer:** Uses native `fetch` (not axios). Two clients:
- `src/services/api.ts` — Auth API client (Node.js backend)
- `src/services/apiUtils.ts` — Generic HTTP client (`ApiUtils` static class) for V4 API

All API responses follow `ServiceResult<T>` pattern: `{ success: boolean; data?: T; error?: string }`.

**Services:** Class-based singletons exported as instances (e.g., `export const themeService = new ThemeService()`). Organized by domain under `src/services/settings/`, `src/services/reports/`, etc.

**State Management:** Zustand stores in `src/store/` with localStorage persistence via `persist` middleware. Key stores: `useModuleStore` (sales/membership toggle), `useThemeStore`, `useReportsStore`, `useVenuesStore`, `useTimezonesStore`.

**Validation:** Zod schemas in `src/validation/`. Validator helper classes wrap `safeParse` and return discriminated results. Validation runs both in stores (before state updates) and in services (before API calls).

**Auth Flow:** Login → `authService.login()` → POST to Node.js API → JWT stored in `localStorage['token']` → token injected as `Bearer` header in all V4 API requests via `ApiUtils`.

**Key Libraries:** `@formio/react` + `formiojs` (form builder), `react-email-editor` (email templates), `recharts` (charts), `@dnd-kit/core` (drag & drop).

### Backend Architecture (keepme-v4)

**Request Flow:** Route → Middleware (AuthMiddleware → CompanyContextMiddleware) → Controller → Action → Repository → Model

**Action Pattern:** Business logic lives in Action classes (`app/Actions/`), not controllers. Actions are injected into controller methods. Base action provides fluent `request()` setter.

**Multi-Tenancy:** JWT token contains company code. `AuthMiddleware` decodes the JWT (firebase/php-jwt, HS256) and sets `X-Company-Id`/`X-User-Id` headers. `CompanyContextMiddleware` uses `KeepmeDatabaseConnectionManager` to dynamically set the database connection based on company code + region (EU/AU/US/GB). Each region has separate MySQL and MongoDB configs in `config/db_connections.php`.

**Database Connections:**
- `master` connection — `keepme_clients` database (users, companies)
- `client` connection — Tenant-specific database (set dynamically per request)
- MongoDB — Activity logs (enabled via `config('logging.enable_mongo_logs')`)

**Models:** All MySQL models extend `BaseModel` (auto-generates UID, sets timestamps, hooks ActivityLogger). Models use `connection = 'client'`. `User` model extends `Authenticatable` instead.

**Repository Pattern:** Interface + implementation bound via `RepositoryServiceProvider`. Base repository provides `find`, `create`, `update`, `delete`, `getLatest`.

**Response Consistency:** `ResponseService` provides standardized methods: `success()`, `error()`, `validationError()`, `notFound()`, `unauthorized()`, `forbidden()`, `paginated()`, `created()`.

**Request Validation:** Form Request classes in `app/Http/Requests/` with `rules()`, `prepareForValidation()`, `withValidator()`, and custom `validated()` that adds audit fields.

**API Documentation:** L5-Swagger/OpenAPI annotations on `BaseController`. Swagger UI available at the API root.

### Message Flow (Webhooks)
```
External Webhook → nginx mirror → km-webhook-service → Kafka → km-webhook-consumer-service → MySQL
```
Kafka topics are tenant-specific: `webhook-{clientName}`

### Antares Data Pipeline
```
Airflow DAGs (antares-data-scraper) → Playwright Service (cookie/token extraction) →
Web Scraping → Data Processing → Storage
```
DAGs organized per-organization (36+ fitness chains). Supports schedule, pricing, membership, staff, and FAQ extraction.

## Infrastructure

- **Orchestration:** Kubernetes on AWS EKS (eu-west-2, us-east-1)
- **Deployment:** ArgoCD for GitOps (auto-healing enabled, 15 retry attempts with exponential backoff)
- **API Gateway:** Kong
- **Cache:** Dragonfly/Redis
- **Message Broker:** Kafka
- **Secret Management:** SOPS with AWS KMS encryption
- **IaC:** Terraform in `keepme-infra/infrastructure/` (modules: vpc, eks, iam, eks-addons)
- **Helm Charts:** `charts/` (root-level, all services) and `keepme-infra/charts/`

### CI/CD (GitHub Actions)
Workflows in each repo: `build.yml` (prod), `build_dev.yml` (dev), `build_staging.yml` (staging)

### Environments
Three environments: `dev`, `staging`, `prod` — each with separate ArgoCD project definitions in `keepme-argo/`.

## Security Requirements

- Run Snyk security scans for new code. Fix reported issues and rescan until clean. This is enforced via Cursor rules in `keepme-react-v4/.cursor/rules/` and `keepme/.cursor/rules/`.
- The `secret/` directory contains service credentials (agent-panel, chatbot, llm, openai-rag). Never commit secrets.
- Root directory contains SSH keys (`keepme_minhlt`, `keepme_server.pem`) — do not modify or commit.

## Database

- **MySQL 8.0:** Primary relational database (multi-tenant, per-company databases)
- **MongoDB 7.0:** Activity logging and document storage
- **Company Mappings:** `keepme-v4/config/company_mappings.php` maps company codes to database names
- **Regional Configs:** `keepme-v4/config/db_connections.php` defines per-region (EU/AU/US/GB) connection details
