# Code Review

Review staged or changed code for quality, security, and correctness.

## Usage
`/project:review`

## Steps

1. Run `git diff` to see all changes
2. Check for security issues (OWASP top 10, secrets, SQL injection, XSS)
3. Check for Snyk-reportable vulnerabilities in new dependencies
4. Verify code follows existing patterns (Action pattern, Repository pattern, ServiceResult<T>)
5. Check test coverage for new logic
6. Summarize findings with severity: critical / warning / suggestion
