# DevSecOps Portfolio – Case Studies & Demonstrated Skills

> Azure Admin / Solutions Architect growing into DevSecOps with hands-on labs, real pipelines, and documented evidence.

---

## Who Is This For?

This portfolio is aimed at:
- **Hiring managers** assessing DevSecOps capability
- **Azure Architects** evaluating depth in security engineering
- **AZ-400 reviewers** mapping experience to exam domains

---

## Skills Demonstrated

| Skill Area | Evidence |
|------------|----------|
| Infrastructure as Code (Bicep) | Module 1 — Landing Zone deployment |
| Policy-as-Code (OPA / Conftest / Checkov) | Modules 1 & 2 — pipeline gates blocking non-compliant IaC |
| SAST / DAST / Secrets Scanning | Module 2 — CodeQL, Semgrep, TruffleHog, OWASP ZAP integrated |
| SBOM & Software Composition Analysis | Module 2 — supply chain security controls |
| Zero-downtime deployment (Blue/Green) | Module 4 — AKS traffic switching |
| Canary releases | Module 5 — ACA percentage-based traffic splitting |
| Observability & Automated IR | Module 3 — App Insights → Log Analytics → Sentinel → Logic Apps |
| Azure DevOps Pipelines | Modules 2–5 — multi-stage YAML pipelines |

---

## Architecture Highlights

### DevSecOps Pipeline (Module 2)

```
  Code Push
      │
      ▼
  ┌─────────────────────────────────────────────────────┐
  │              Azure DevOps Pipeline                  │
  │                                                     │
  │  Stage 1: Build                                     │
  │    ├── CodeQL  (SAST — finds code vulnerabilities)  │
  │    ├── Semgrep (SAST — custom security rules)       │
  │    └── TruffleHog (Secrets — API keys, passwords)   │
  │                                                     │
  │  Stage 2: IaC Scan                                  │
  │    ├── Checkov  (Bicep / Terraform misconfigs)      │
  │    └── Conftest / OPA (custom policy gates)         │
  │                                                     │
  │  Stage 3: Test                                      │
  │    └── OWASP ZAP (DAST — attacks running app)       │
  │                                                     │
  │  Stage 4: Deploy (manual approval gate)             │
  │    └── Bicep / Helm → Azure                         │
  └─────────────────────────────────────────────────────┘
```

### Observability Stack (Module 3)

```
  App / Infra Events
        │
        ▼
  Application Insights ──► Log Analytics Workspace
                                    │
                                    ▼
                            Azure Sentinel
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                    Alert Rules           Logic Apps
                                     (Auto-remediation)
```

---

## Impact Statements

- Reduced deployment risk by enforcing **policy-as-code gates** that block non-compliant infrastructure before it reaches Azure
- Eliminated manual security reviews by embedding **SAST, DAST, and secrets scanning** directly in CI/CD
- Delivered **SBOM + SCA** outputs on every build to meet supply chain compliance requirements
- Achieved **zero-downtime releases** using Blue/Green on AKS and Canary on Azure Container Apps
- Built **automated incident response** workflows: Sentinel raises an incident → Logic App notifies team and isolates resource

---

## Repositories

- `devsecops-practice` (this repo) — training, reference architecture, and labs
- Add links to any companion repos here (e.g., a real pipeline project, a Bicep module library)

---

## Proof Screenshots

Drop pipeline run screenshots, Defender findings, Sentinel incidents, and deployment logs in `/proof/`.

Suggested filenames:
```
proof/
├── m1-bicep-deploy.png
├── m2-codeql-findings.png
├── m2-checkov-gate.png
├── m3-sentinel-incident.png
├── m4-aks-bluegreen-switch.png
└── m5-aca-canary-traffic.png
```

---

## AZ-400 Alignment

| AZ-400 Exam Domain | Evidence |
|--------------------|---------|
| Design & implement source control | Branch policies, PR templates |
| Design & implement build/release pipelines | Modules 2–5 |
| Security & compliance plan | Modules 1–2 |
| Instrumentation strategy | Module 3 |
| Deployment strategies | Modules 4–5 |

> Exam registration: [AZ-400 on Microsoft Learn](https://learn.microsoft.com/en-us/credentials/certifications/devops-engineer/)
