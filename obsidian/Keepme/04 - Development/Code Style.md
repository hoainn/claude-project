# Code Style & Conventions

## Backend (PHP/Laravel)

- **Action classes** for all business logic (single responsibility)
- **Repository pattern** with interfaces bound in `RepositoryServiceProvider`
- **ResponseService** for all API responses
- **BaseModel** for all Eloquent models (UID, timestamps, activity logging)
- **Request classes** for validation (`rules()` + `prepareForValidation()`)
- Use `connection = 'client'` on models for multi-tenant queries
- Secrets via env vars + SOPS — never hardcoded

## Frontend (React/TypeScript)

- **ApiUtils** static class for all HTTP requests
- **ServiceResult\<T\>** type for consistent responses
- **Zustand stores** with localStorage persistence
- **Zod schemas** for form validation
- **Service classes** exported as singletons
- **Custom hooks** for component logic extraction
- **CSS modules** for scoped styling

## Microservices (Fastify/TypeScript)

- **Fastify plugins** for modular setup
- **Awilix** for DI container
- **Vitest** for tests
- **Knex** for DB migrations
- **KafkaJS** for event streaming

## General

- No secrets committed
- Security scans (Snyk) before commit
- Concise code — no unnecessary abstractions
- No error handling for impossible scenarios
