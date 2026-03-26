# Backend API (keepme-v4)

## Stack
- Laravel 12, PHP 8.3
- MySQL 8 (multi-tenant, per company code)
- MongoDB 7 (per-tenant)
- Redis (queues + caching)

## Directory Structure

```
app/
├── Actions/        # Business logic (180+ classes)
├── Models/         # Eloquent models (170+)
├── Repositories/   # Data access layer (130+)
├── Services/       # Domain services (78+)
├── Http/
│   ├── Controllers/  # API endpoints (53+)
│   ├── Requests/     # Validation (47+)
│   ├── Resources/    # Response transformers (39+)
│   └── Middleware/   # HTTP middleware (10+)
├── Jobs/           # Queue jobs (9+)
├── Enums/          # PHP enums (31+)
├── Observers/      # Model observers (11+)
├── Policies/       # Authorization (24+)
└── Traits/         # Shared traits (8+)
```

## Key Patterns

### Action Classes
Single responsibility business logic. Each action = one operation.
```php
class GetLeadAction extends ActionBase { ... }
class CreateBatchWhatsappCampaignAction extends ActionBase { ... }
```

### Repository Pattern
Interface-based. Bound in `RepositoryServiceProvider`.
```php
interface AccountRepositoryInterface { ... }
class AccountRepository implements AccountRepositoryInterface { ... }
```

### BaseModel
All models extend `BaseModel`:
- Auto-generated UIDs
- Automatic timestamps
- Activity logging
- `connection = 'client'` for multi-tenant queries

### API Response Format
```json
{
  "success": true,
  "data": {},
  "message": "",
  "error": "",
  "meta": { "current_page": 1, "total": 100 },
  "links": { "first": "...", "last": "...", "prev": null, "next": "..." }
}
```

All responses via `ResponseService`.

## Multi-Tenancy

1. JWT contains company code (e.g. `V4EU`, `AU`)
2. Middleware reads code → selects DB connection
3. `config/db_connections.php` maps codes → MySQL/MongoDB configs
4. No manual connection management needed in controllers

## Queue System
- Driver: Redis
- Runs: `php artisan queue:work --verbose --tries=3 --timeout=90`
- Scheduler: `php artisan schedule:run` every minute

## Commands

```bash
docker-compose up -d
docker-compose exec api php artisan migrate
docker-compose exec api php artisan test
docker-compose exec api php artisan queue:restart
# API: http://localhost:8080
```

> Tests use SQLite `:memory:` — never against real DB
