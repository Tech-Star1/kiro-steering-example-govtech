Enter your requirements in natural language, including compliance mandates, functional requirements, and security controls. 

The following example shows a spec input for a citizen inquiry system:

Build a citizen inquiry system for our state government agency that handles 
questions from residents across multiple channels (web portal, email, phone 
transcripts).

COMPLIANCE REQUIREMENTS:
- Section 508 accessibility (WCAG 2.1 Level AA) for all citizen-facing interfaces
- State Data Protection Act compliance for all citizen PII
- 7-year audit trail retention for all citizen interactions
- Federal Information Processing Standards (FIPS) 140-2 validated encryption for data at rest and in transit
- FedRAMP Moderate authorized AWS services only

FUNCTIONAL REQUIREMENTS:
- Natural language processing to understand and categorize citizen questions
- Integration with existing case management system (legacy SOAP API)
- Mobile-responsive citizen portal
- Staff dashboard for inquiry management and reporting

SECURITY REQUIREMENTS:
- Role-based access control (citizen, staff, supervisor, admin)
- Multi-factor authentication for all staff access
- Session timeout after 15 minutes of inactivity
- All API calls logged to AWS CloudTrail

