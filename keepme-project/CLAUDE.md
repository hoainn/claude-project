# KeepMe

Multi-service SaaS platform for fitness/wellness facility management.

## Repositories

| Directory | Description | Stack |
|-----------|-------------|-------|
| `keepme-react-v4/` | Frontend (latest) | React 19.1, TypeScript 5.8, Vite 7.1 |
| `keepme-v4/` | Backend API (latest) | Laravel 12, PHP 8.3, MySQL 8, MongoDB 7 |
| `keepme/` | Legacy backend | Laravel 8, PHP 7.3/8.0 |
| `keepme-services/` | Webhook microservices | Fastify 5, TypeScript, Kafka |
| `keepme-infra/` | Infrastructure as Code | Terraform, Kubernetes, Helm |
| `keepme-argo/` | ArgoCD GitOps | Kubernetes manifests |
| `antares-data-scraper/` | Airflow data extraction | Python, Airflow |
| `antares-playwrite-service/` | Browser automation | Python, FastAPI, Playwright |
| `antares-project/` | Multi-service automation | Node.js, Python, MongoDB, React |

## Commands

### Frontend (keepme-react-v4)
```bash
npm run dev          # Dev server on port 5173
npm run build        # tsc -b && vite build
npm run lint         # ESLint
```

### Backend API (keepme-v4)
```bash
docker-compose up -d
docker-compose exec api php artisan migrate
docker-compose exec api php artisan test
docker-compose exec api php artisan queue:restart
# API: http://localhost:8080
```

### Microservices (keepme-services/src/km-webhook-service)
```bash
npm run test         # Vitest
npm run build        # Compile TypeScript
```

### Infra (keepme-argo)
```bash
# All connector changes via GitOps — edit values.yaml, commit, push, then:
kubectl annotate application <app-name> -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

## Non-obvious gotchas

- **Backend tests** use SQLite `:memory:` — run `php artisan test`, not against real DB
- **Frontend uses two APIs**: Node.js (auth only) + V4 (everything else) — see `src/config/api.config.ts`
- **Multi-tenancy**: company code in JWT → dynamic DB connection per request (`config/db_connections.php`)
- **SSH key for git push**: `GIT_SSH_COMMAND="ssh -i /Users/hoainn/Documents/Project/Keepme/keepme_minhlt"`
- **Never commit**: `secret/`, `keepme_minhlt`, `keepme_server.pem`, `.env` files
- **Kafka commands**: always pass `env JMX_PORT="" KAFKA_JMX_OPTS=""` or they fail
- **Redshift views**: always update in `keep-me` database, not `dev` (dev points to non-existent Glue DB)

## Rules & agents

Detailed instructions are in `.claude/rules/` — loaded automatically:
- `rules/code-style.md` — backend/frontend/microservice patterns
- `rules/infrastructure.md` — K8s, Kafka, ArgoCD, data pipeline
- `rules/data-warehouse.md` — Glue, Redshift, Kafka Connect, Parquet
- `rules/preferences.md` — working preferences, connector reset order
