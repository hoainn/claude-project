---
description: Trigger ArgoCD refresh after a git push to keepme-argo
argument-hint: <app-name>
---

## App: $ARGUMENTS

## Recent keepme-argo commits
!`cd keepme-argo 2>/dev/null && git log --oneline -3`

## Current ArgoCD app status
!`kubectl get application $ARGUMENTS -n argocd -o jsonpath='{.status.sync.status}/{.status.health.status}' 2>/dev/null`

Trigger ArgoCD sync for `$ARGUMENTS` and watch until healthy:
```bash
kubectl annotate application $ARGUMENTS -n argocd argocd.argoproj.io/refresh=normal --overwrite
```
