# Redactable

Document redaction platform — multi-service SaaS.

## Repositories

| Directory | Description | Stack |
|-----------|-------------|-------|
| `redactable-frontend-v2/` | Document redaction UI | Angular 20, TypeScript, NgRx |
| `redactable/` | Backend REST API | .NET 7, MSSQL, Redis, SignalR |
| `redactable-ocr/` | OCR processing service | Python, PaddlePaddle, Tesseract |
| `redactable-ui-library/` | Shared component library | Angular, Storybook |
| `batch-redaction-frontend/` | Batch processing UI | Angular 20, Nx, Native Federation |
| `redaction-library/` | Core redaction logic | TypeScript |
| `redactable-byoc/` | Bring Your Own Cloud | — |
| `redactable-enterprise-infra/` | Infrastructure as Code | Terraform, AWS EKS, Helm |
| `redactable-argo-apps/` | GitOps application definitions | ArgoCD |
| `redactable-charts/` | Kubernetes Helm charts | Helm (Wazuh security) |
| `redactable-security/` | Security analysis tools | AWS Inspector analysis |

## Commands

### Frontend (redactable-frontend-v2)
```bash
nvm use && npm i
npm start                   # Dev server at localhost:4200
npm run build:prod          # Production build
npm run test:local          # Jest unit tests
npm run cy:open             # Cypress E2E UI
npm run generate-api-client # Regenerate from Swagger
```

### Backend (redactable)
```bash
docker-compose -f docker-compose.services.yml up   # Start MSSQL + Redis
dotnet run --project redactable-backend/Redactable.API/
# Swagger: http://localhost:5000/swagger/index.html
```

### OCR Service (redactable-ocr)
```bash
git lfs install && git lfs pull   # Requires git-lfs for ONNX models
docker build --tag ocr .
docker run --network=host --gpus all -it ocr
```

### Batch Frontend (batch-redaction-frontend)
```bash
npm start    # Dev server at localhost:4201
npm test     # Vitest unit tests
```

### Infrastructure (redactable-enterprise-infra)
```bash
ENV=prod make init    # Initialize S3 backend
ENV=prod make plan    # Preview changes
ENV=prod make apply   # Apply changes
```

## Non-obvious gotchas

- **SSH key for git push**: `GIT_SSH_COMMAND="ssh -i /Users/hoainn/Documents/Project/Redactable/quang-redactable"`
- **Never commit**: `quang-redactable`, `*.pem`, `.env` files
- **OCR models**: require `git lfs pull` — large ONNX files not in regular clone
- **Frontend APIs**: Angular client is regenerated via `npm run generate-api-client` from Swagger
- **State management**: NgRx functional approach (`createFeature`, `createReducer`) — no class-based actions
- **GitOps**: deploy via ArgoCD — edit values.yaml, commit, push; do NOT apply manifests directly
- **Clusters**: hub-cluster (ArgoCD/monitoring), uat-cluster, prod-cluster

## Rules & agents

Detailed instructions in `.claude/rules/` — loaded automatically:
- `rules/code-style.md` — Angular/TypeScript, C#, CSS, testing patterns
- `rules/infrastructure.md` — AWS EKS, Terraform, ArgoCD, Helm
- `rules/preferences.md` — working preferences
