# Working Preferences

## Responses
- Concise responses, no trailing summaries
- Skip explaining obvious steps

## Planning
- Default to executing directly — do not plan unless the task is large, multi-system, and has irreversible steps

## Code Generation
- Do not add docstrings, comments, or type annotations to code that wasn't changed
- Do not add error handling or validation for scenarios that can't happen
- Do not create helpers or abstractions for one-time operations
- Avoid backwards-compatibility hacks

## Frontend (Angular)
- Always use new Angular control flow (`@if`, `@for`) — never `*ngIf`, `*ngFor`
- Always use `inject()` — never constructor injection
- Always use `input()` / `output()` — never `@Input()` / `@Output()`
- Regenerate API client after backend changes: `npm run generate-api-client`

## Infrastructure
- Never apply Kubernetes manifests directly — always via ArgoCD GitOps
- Never run Terraform apply without reviewing plan first
- Do NOT commit SSH keys, `.env` files, or `*.pem` files
