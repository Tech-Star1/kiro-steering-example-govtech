# GovTech Security Standards

## Security Requirements
- All citizen data encrypted at rest (AES-256) and in transit (TLS 1.2+)
- FIPS 140-2 validated encryption modules required
- Multi-factor authentication required for all staff access
- Role-based access control with principle of least privilege
- Session timeout: 15 minutes of inactivity
- All API calls logged to AWS CloudTrail
- PII must be redacted in application logs
- Security scanning in CI/CD pipeline (OWASP Top 10 checks)

## AWS Service Requirements
- Use only FedRAMP Moderate authorized services
- Deploy in US East (N. Virginia) or US West (Oregon) regions
- Enable AWS CloudTrail logging for all API calls
- Use AWS KMS for encryption key management
- Tag all resources with: Department, CostCenter, Environment, DataClassification

## State Data Protection Requirements
- Comply with state-specific data protection laws
- Data residency requirements: citizen data must remain within US boundaries
- Data retention policies must align with state records management guidelines
- Right to access and deletion requests must be supported

## Section 508 Accessibility Standards
- All web applications must meet WCAG 2.1 AA compliance
- Screen reader compatibility required for all public-facing interfaces
- Keyboard navigation must be fully supported
- Color contrast ratios must meet accessibility thresholds (4.5:1 minimum)
- Alternative text required for all images and media

## Audit Logging Patterns
- Log all authentication events (success and failure)
- Log all data access events involving PII or sensitive data
- Log all administrative actions and configuration changes
- Retain audit logs for minimum 7 years per state requirements
- Logs must be immutable and tamper-evident

## Security Scanning Thresholds
- No critical vulnerabilities allowed in production deployments
- High severity vulnerabilities must be remediated within 30 days
- Medium severity vulnerabilities must be remediated within 90 days
- Static Application Security Testing (SAST) required in CI/CD
- Dynamic Application Security Testing (DAST) required before production release
- Software Composition Analysis (SCA) for dependency vulnerabilities
