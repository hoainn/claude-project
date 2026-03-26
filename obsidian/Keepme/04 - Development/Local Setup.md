# Local Setup

## Frontend (keepme-react-v4)

```bash
cd keepme-react-v4
npm install
npm run dev      # http://localhost:5173
```

## Backend (keepme-v4)

```bash
cd keepme-v4
docker-compose up -d
docker-compose exec api php artisan migrate
# API: http://localhost:8080
```

Docker services:
- PHP API on port 8080 (nginx reverse proxy)
- MySQL on port 3306
- MongoDB on port 27017
- Scheduler: `php artisan schedule:run` every 60s
- Queue: `php artisan queue:work --verbose --tries=3 --timeout=90`

## Microservices (keepme-services)

```bash
cd keepme-services/src/km-webhook-service
npm install
npm run build
npm run start
```

## Git SSH Key

```bash
GIT_SSH_COMMAND="ssh -i /Users/hoainn/Documents/Project/Keepme/keepme_minhlt" git push
```

## Never Commit

- `secret/`
- `keepme_minhlt`
- `keepme_server.pem`
- `.env` files
