# Code Style Rules

## Backend (keepme-v4 — Laravel 12, PHP 8.3)
- Business logic in Action classes (`app/Actions/`), not controllers
- Use Repository pattern — interface + implementation bound via `RepositoryServiceProvider`
- All responses via `ResponseService` (success, error, validationError, paginated, etc.)
- Models extend `BaseModel` (auto UID, timestamps, ActivityLogger); use `connection = 'client'`
- Form validation in `app/Http/Requests/` with `rules()`, `prepareForValidation()`, `withValidator()`

## Frontend (keepme-react-v4 — React 19, TypeScript 5.8)
- API calls via `ApiUtils` static class (V4) or `api.ts` (auth/Node.js)
- All API responses follow `ServiceResult<T>`: `{ success: boolean; data?: T; error?: string }`
- Services are class-based singletons exported as instances
- Validation with Zod schemas in `src/validation/`
- State in Zustand stores (`src/store/`) with localStorage persistence

## Microservices (keepme-services — Fastify 5, TypeScript)
- Kafka topics are tenant-specific: `webhook-{clientName}`
- Tests use Vitest

## General
- No secrets committed — use SOPS + AWS KMS
- Run Snyk security scans on new code; fix all issues before committing
- Do not modify SSH keys (`keepme_minhlt`, `keepme_server.pem`) in the root directory
