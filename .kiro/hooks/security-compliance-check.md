# Security Compliance Check Hook

## Trigger
When code is committed

## Instructions
Verify security and audit compliance against GovTech standards:

### Security Checks
- All citizen PII encrypted at rest (AES-256) and in transit (TLS 1.2+)
- Audit logging present for all citizen data access and modifications
- Authentication required for all administrative functions
- Role-based access control implemented correctly
- Session timeout configured (15 minutes)
- Error messages don't expose sensitive information
- Database queries use parameterized statements (SQL injection prevention)
- API endpoints include rate limiting
- Secrets not hardcoded in source code

### Output
Generate a security compliance report highlighting any concerns.

For each issue found:
1. Identify the file and line number
2. Describe the security concern
3. Rate severity (Critical/High/Medium/Low)
4. Suggest specific fixes based on our GovTech security standards

### Report Format
```
## Security Compliance Report
**Scan Date:** [date]
**Files Scanned:** [count]
**Status:** [PASS/FAIL]

### Issues Found
| Severity | File | Line | Issue | Recommended Fix |
|----------|------|------|-------|-----------------|

### Summary
- Critical: [count]
- High: [count]
- Medium: [count]
- Low: [count]

### Compliance Status
- [ ] Encryption requirements met
- [ ] Audit logging implemented
- [ ] Authentication enforced
- [ ] RBAC implemented
- [ ] Session management configured
- [ ] Error handling secure
- [ ] SQL injection prevented
- [ ] Rate limiting configured
- [ ] No hardcoded secrets
```
