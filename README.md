# 🏛️ Kiro Artifacts: GovTech

> Companion repository for the AWS Public Sector blog post **"Accelerating GovTech Development with Kiro"**

Build compliant government applications faster. This repo contains everything from the blog walkthrough: a complete citizen inquiry system spec, compliance-focused steering files, and automated hooks for security, accessibility, and audit documentation.

---

## 🧩 What Are Kiro Artifacts?

[Kiro](https://kiro.dev) is AWS's AI-powered IDE that helps developers build applications faster. But Kiro gets smarter when you give it context about your project, your standards, and your workflows. That's where **Kiro artifacts** come in.

This repository contains three types of artifacts that make Kiro more effective for GovTech development:

| Artifact Type | Location | Purpose |
|---------------|----------|---------|
| **Steering Files** | `.kiro/steering/` | Markdown docs describing your compliance standards, security requirements, and coding conventions. Kiro references these when generating code. |
| **Agent Hooks** | `.kiro/hooks/` | Automated actions that trigger during development — security checks on save, accessibility validation, audit doc generation. |
| **Specs** | `.kiro/specs/` | Structured requirements, design, and task documents that Kiro uses for spec-driven development. |
| **MCP Config** | `.kiro/settings/mcp.json` | Model Context Protocol servers that connect Kiro to external tools, APIs, and documentation. |

Clone this repo, open it in Kiro, and these artifacts activate automatically. Use them as-is for GovTech projects, or customize them for your organization's specific compliance requirements.

---

## 🚀 Quick Start

```bash
# 1. Install Kiro from kiro.dev
# 2. Clone and open
git clone https://github.com/Tech-Star1/kiro-config-example-govtech.git
cd kiro-config-example-govtech
kiro .
```

Steering files and hooks activate automatically. Start building with compliance baked in.

---

## 📦 What's Inside

### Spec-Driven Development
Complete specification for a citizen inquiry system → `.kiro/specs/citizen-inquiry-system/`

| File | What It Does |
|------|--------------|
| `requirements.md` | Functional, compliance, security, and performance requirements |
| `design.md` | FedRAMP-compliant architecture using authorized AWS services |
| `tasks.md` | Implementation roadmap with compliance checkpoints |

### Steering Files
Your compliance standards, codified → `.kiro/steering/`

| File | Coverage |
|------|----------|
| `govtech-security-standards.md` | AES-256 encryption, TLS 1.2+, FIPS 140-2, FedRAMP Moderate services, Section 508, audit logging |

### Agent Hooks
Automated compliance checking that runs while you code → `.kiro/hooks/`

| Hook | Trigger | What It Catches |
|------|---------|-----------------|
| `security-compliance-check` | File save | PII encryption, audit logging, RBAC, session timeouts, SQL injection, hardcoded secrets |
| `accessibility-508-check` | UI file save | Keyboard nav, form labels, alt text, color contrast, ARIA attributes, focus indicators |
| `ferpa-audit-docs` | Agent completion | API changes, data access patterns, security controls → audit-ready documentation |

### MCP Integration
Connect Kiro to external tools and documentation → `.kiro/settings/mcp.json`

```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"],
      "disabled": false,
      "autoApprove": []
    },
    "aws-docs": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": { "FASTMCP_LOG_LEVEL": "ERROR" },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

**Requires:** `uvx` from the [uv package manager](https://docs.astral.sh/uv/getting-started/installation/)

---

## 🛡️ Compliance Frameworks

| Framework | Scope |
|-----------|-------|
| **FedRAMP** | Federal cloud security (Moderate baseline) |
| **FISMA** | Federal information security |
| **CJIS** | Criminal justice information |
| **FERPA** | Student data privacy |
| **Section 508** | Accessibility (WCAG 2.1 AA) |
| **FIPS 140-2** | Encryption validation |
| **State Laws** | State-specific data protection |

---

## 🔧 Customize for Your Org

### Add Steering Files
Drop markdown files in `.kiro/steering/` with your standards:

```markdown
# Your Agency Security Standards

## Encryption
- All PII encrypted at rest (AES-256) and in transit (TLS 1.2+)
- FIPS 140-2 validated modules required

## AWS Services
- FedRAMP Moderate authorized only
- US East or US West regions
- CloudTrail logging on all API calls
```

### Add Hooks
Create `.kiro.hook` files in `.kiro/hooks/`:

```
When code is committed, verify:
- PII encryption (AES-256 at rest, TLS 1.2+ in transit)
- Audit logging for all data access
- No hardcoded secrets
- Parameterized SQL queries

Generate a compliance report with specific fixes.
```

### CI/CD Integration

```yaml
# GitHub Actions
compliance-check:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Security check
      run: kiro chat "Review for security compliance"
    - name: Accessibility check
      run: kiro chat "Check UI for Section 508 compliance"
```

---

## 📁 Repo Structure

```
.kiro/
├── hooks/
│   ├── accessibility-508-check.kiro.hook
│   ├── ferpa-audit-docs.kiro.hook
│   ├── security-compliance-check.kiro.hook
│   └── security-compliance-check.md
├── settings/
│   └── mcp.json
├── specs/
│   └── citizen-inquiry-system/
│       ├── design.md
│       ├── requirements.md
│       └── tasks.md
└── steering/
    └── govtech-security-standards.md
```

---

## 🔗 Resources

- [Kiro](https://kiro.dev) — IDE and CLI
- [Kiro Enterprise](https://kiro.dev/enterprise/) — Customer managed keys, regional storage, content privacy
- [Kiro Docs](https://kiro.dev/docs)
- [AWS Public Sector](https://aws.amazon.com/government-education/)
- [AWS Compliance Programs](https://aws.amazon.com/compliance/services-in-scope/)

---

## ✍️ Author

**Warner Bell** — Technical Account Manager, AWS Public Sector

Specializing in AI/ML implementations and cloud modernization for state and local government organizations and educational institutions. Expertise in government compliance frameworks including FedRAMP, CJIS, and FISMA. Seven AWS certifications including Solutions Architect Professional. AI/ML TFC Ambassador.

---

## 📄 License

MIT-0 — See [LICENSE](LICENSE)
