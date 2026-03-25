---
description: Review current branch changes for quality, security, and correctness
---

## Changed files
!`git diff --name-only main...HEAD 2>/dev/null || git diff --name-only HEAD~1...HEAD`

## Full diff
!`git diff main...HEAD 2>/dev/null || git diff HEAD~1...HEAD`

Review the above changes for:
1. Security vulnerabilities (OWASP top 10, secrets, SQL injection, XSS)
2. Code pattern adherence (Action pattern, Repository pattern, ServiceResult<T>)
3. Missing test coverage for new logic
4. Performance concerns

Give specific, actionable feedback per file. Severity: **critical** / **warning** / **suggestion**.
