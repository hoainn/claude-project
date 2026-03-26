# Testing

## Backend (keepme-v4)

```bash
docker-compose exec api php artisan test
```

- Framework: PHPUnit
- Database: SQLite `:memory:` — **NOT** against real DB
- Clear cache before testing: `php artisan config:clear`

## Frontend (keepme-react-v4)

```bash
npm test
npm run test:coverage
```

- Framework: Vitest
- Test files: `*.test.ts`, `*.spec.ts`

## Microservices (keepme-services)

```bash
npm run test
npm run test:coverage
npm run test:watch
```

- Framework: Vitest

## Code Quality

| Tool | Purpose | Repo |
|------|---------|------|
| PHPStan | Static analysis | keepme-v4 |
| PHP CS Fixer | Code formatting | keepme-v4 |
| ESLint | Linting | keepme-react-v4 |
| Prettier | Formatting | keepme-react-v4 |
| Snyk | Security scan | All |
