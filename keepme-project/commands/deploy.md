---
description: Deploy a service to dev/staging/prod via ArgoCD GitOps
argument-hint: <service> <environment>
---

## Deploy: $ARGUMENTS

## Current branch & uncommitted changes
!`git branch --show-current && git status --short`

## Recent commits
!`git log --oneline -5`

## Target ArgoCD apps
!`kubectl get application -n argocd 2>/dev/null | grep "$(echo "$ARGUMENTS" | awk '{print $1}')" | head -5`

Deploy `$ARGUMENTS` following GitOps:
1. Confirm branch is clean and changes are committed
2. Update image tag or values in `keepme-argo/$(echo "$ARGUMENTS" | awk '{print $2}')/`
3. Commit and push to keepme-argo
4. Trigger ArgoCD sync and monitor rollout
