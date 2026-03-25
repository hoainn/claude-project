---
description: Debug a broken data pipeline for a specific collection
argument-hint: <collection-name>
---

## Pipeline debug: $ARGUMENTS

## Pod status
!`kubectl get pods -n data-warehouse-prod | grep "$ARGUMENTS" 2>/dev/null`

## Recent error logs
!`kubectl logs -n data-warehouse-prod -l "app=source-$ARGUMENTS" --tail=50 2>/dev/null | grep -E "(ERROR|FAILED|Exception|Caused by)" | tail -10`

## Schema Registry subjects
!`curl -s http://10.200.95.52:8081/subjects 2>/dev/null | python3 -c "import sys,json; [print(s) for s in json.load(sys.stdin) if '$ARGUMENTS'.replace('-','').lower() in s.lower()]" 2>/dev/null`

Debug the pipeline for `$ARGUMENTS`:
1. Check pod status above — is task RUNNING or FAILED?
2. If FAILED: diagnose from error logs above
3. Check Schema Registry for type conflicts across all versions
4. Compare Schema Registry vs Glue table columns — fix Glue if needed
5. Re-register S3 partitions if table was recreated
6. Rebuild `cdc.$ARGUMENTS` view in `keep-me` database
