---
layout: page
title: Learning Path
permalink: /learning-path
---

# AZ-400 Learning Path — 4-Week Plan

> Designed for self-learners who prefer **doing over reading**. Each week has a hands-on lab first, then the theory to explain what you just built.

---

## How Each Week Works

```
  ┌─────────────────────────────────────────────────┐
  │  Each Week                                      │
  │                                                 │
  │  Day 1-2: BUILD the lab (hands-on)              │
  │  Day 3:   READ the module doc (understand why)  │
  │  Day 4:   FLASHCARDS (memorise key terms)       │
  │  Day 5:   QUIZ (test yourself)                  │
  │  Day 6-7: Review gaps, re-do weak areas         │
  └─────────────────────────────────────────────────┘
```

---

## Week 1 — Azure Landing Zone & IaC

**Goal:** Understand how enterprise Azure environments are structured and governed before any workload is deployed.

### What you will build
- A Management Group hierarchy (Root → Platform → Landing Zones → Sandboxes)
- A Bicep template that deploys a resource group with tags and locks
- An Azure Policy assignment that denies resources without required tags
- A Conftest policy gate that checks your Bicep before it deploys

### Visual — Azure Landing Zone Structure

```
  Root Management Group
  │
  ├── Platform
  │   ├── Management     (Log Analytics, Sentinel)
  │   ├── Connectivity   (Hub VNet, Firewall, DNS)
  │   └── Identity       (AD DS, RBAC)
  │
  └── Landing Zones
      ├── Corp            (Internal workloads)
      └── Online          (Internet-facing apps)
          │
          └── Sandbox     (Dev/Test — less strict policy)
```

### Lab Steps
1. In the Azure Portal, create a Management Group under your root
2. Deploy a Bicep file that creates a resource group — add a tag `environment: dev`
3. Assign the built-in Azure Policy `Require a tag on resource groups`
4. Write a simple Conftest `.rego` rule that fails if `environment` tag is missing
5. Run `conftest test bicep-output.json` — watch it pass and fail

### Key Concepts

| Term | Plain-English Meaning |
|------|-----------------------|
| Management Group | A folder for Azure subscriptions — apply policy to all subscriptions at once |
| Azure Policy | A rule that Azure enforces automatically (deny, audit, modify) |
| Bicep | Microsoft's IaC language — cleaner than ARM JSON, compiles to ARM |
| Conftest | A tool that checks config files against OPA (Open Policy Agent) rules |
| Policy-as-Code (PaC) | Writing your security rules as code that runs in your pipeline |

### AZ-400 Exam Tips
- Know the difference between **initiative** (group of policies) and **policy definition**
- Understand **effect types**: Deny, Audit, AuditIfNotExists, DeployIfNotExists, Modify
- Bicep has first-class support in Azure DevOps pipelines via the `AzureCLI` task

### References
- [Microsoft Cloud Adoption Framework – Landing Zones](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure Policy documentation](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Conftest / OPA](https://www.conftest.dev/)
- [Enterprise-Scale reference architecture (GitHub)](https://github.com/Azure/Enterprise-Scale)

---

## Week 2 — DevSecOps CI/CD Pipeline & Security Gates

**Goal:** Build a multi-stage pipeline that automatically scans code, IaC, and secrets before anything reaches Azure.

### What you will build
- An Azure DevOps pipeline with 4 stages: Build → IaC Scan → DAST → Deploy
- CodeQL or Semgrep SAST scan integrated into the build stage
- TruffleHog secrets scan on every commit
- Checkov IaC scan blocking deployment on failures
- OWASP ZAP DAST scan against a test deployment

### Visual — Shift-Left Security

```
  Traditional approach:
  Code → Build → Test → Deploy → THEN security review
                                        ↑ Too late!

  Shift-Left approach (what we build):
  Code → [SAST] → Build → [IaC Scan] → Deploy → [DAST]
           ↑                   ↑                    ↑
      Find bugs           Block bad            Attack the
      in source           infra config         running app
```

### Lab Steps
1. Create an Azure DevOps project and push sample code
2. Add a `.github/workflows` or `azure-pipelines.yml` with 4 stages
3. Add CodeQL action/task to Stage 1
4. Add Checkov task to Stage 2 — set `soft_fail: false` to block on HIGH findings
5. Add TruffleHog to pre-commit hook and pipeline
6. Deploy a test app and run OWASP ZAP baseline scan against it
7. Deliberately introduce a secret in code — confirm TruffleHog catches it

### Key Concepts

| Term | Plain-English Meaning |
|------|-----------------------|
| SAST | Static Application Security Testing — reads your source code for vulnerabilities |
| DAST | Dynamic Application Security Testing — attacks your running application |
| Secrets Scanning | Finds API keys, passwords, tokens committed to source control |
| IaC Scanning | Checks Bicep/Terraform for misconfigurations (open ports, no encryption) |
| Policy Gate | A pipeline step that **fails the build** if a security check doesn't pass |
| SBOM | Software Bill of Materials — a list of every package your app depends on |
| SCA | Software Composition Analysis — checks if those packages have known CVEs |

### AZ-400 Exam Tips
- Know the **order of security checks** in a DevSecOps pipeline (SAST before DAST)
- Understand **branch protection rules** — require PR reviews + status checks to pass
- Know what **approval gates** are in Azure DevOps release pipelines
- Checkov maps findings to **CIS benchmarks** and **Azure Security Benchmark**

### References
- [DevSecOps on Azure (Microsoft)](https://learn.microsoft.com/en-us/azure/devops/devsecops/)
- [CodeQL documentation](https://codeql.github.com/docs/)
- [Checkov IaC scanner](https://github.com/bridgecrewio/checkov)
- [TruffleHog secrets scanner](https://github.com/trufflesecurity/trufflehog)
- [OWASP ZAP](https://www.zaproxy.org/)
- [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)

---

## Week 3 — Observability & Incident Response

**Goal:** Know what is happening in your Azure environment at all times, and automate the first response when something goes wrong.

### What you will build
- Application Insights connected to a sample app (traces, metrics, exceptions)
- A Log Analytics workspace with diagnostic settings from key resources
- An Azure Monitor alert rule that fires when error rate exceeds a threshold
- A Sentinel workspace with analytics rules
- A Logic App that auto-notifies a Teams channel when Sentinel raises an incident

### Visual — Observability Pipeline

```
  Your App                 Your Infrastructure
      │                           │
      ▼                           ▼
  App Insights            Diagnostic Settings
  (traces, metrics,       (activity logs, metrics,
   exceptions, deps)       NSG flow logs)
      │                           │
      └─────────────┬─────────────┘
                    ▼
           Log Analytics Workspace
           (central data lake for logs)
                    │
           ┌────────┴────────┐
           ▼                 ▼
      Azure Monitor      Azure Sentinel
      (metric alerts,    (threat detection,
       log alerts)        SIEM + SOAR)
           │                 │
           ▼                 ▼
      Action Group      Logic App
      (email/SMS)       (auto-remediation)
```

### Lab Steps
1. Create a Log Analytics workspace and connect App Insights to it
2. Add Application Insights SDK to a sample app (or use Azure App Service)
3. Create a diagnostic setting on an Azure Key Vault — send logs to LA workspace
4. Create an alert rule: fire when `requests/failed > 5 in 5 minutes`
5. Enable Microsoft Sentinel on the LA workspace
6. Add the `Azure Activity` data connector
7. Create an analytics rule: alert on `CRITICAL` severity security events
8. Build a Logic App that posts to a Teams webhook when an incident is created

### Key Concepts

| Term | Plain-English Meaning |
|------|-----------------------|
| Application Insights | Azure's APM tool — tracks every request, exception, dependency call |
| Log Analytics | A central database for all your Azure logs — query with KQL |
| KQL | Kusto Query Language — SQL-like language for querying Azure logs |
| Azure Monitor | Umbrella service for alerts, metrics, and dashboards |
| Azure Sentinel | Microsoft's SIEM — correlates signals and raises security incidents |
| SOAR | Security Orchestration, Automation and Response — automate incident handling |
| Logic App | Azure's workflow automation — like Power Automate for Azure |

### AZ-400 Exam Tips
- Know the difference between **metric alerts** (near real-time) and **log alerts** (query-based)
- Understand **distributed tracing** in Application Insights — correlation IDs across microservices
- KQL is tested — practice `project`, `where`, `summarize`, `extend`, `join`
- Sentinel uses **MITRE ATT&CK framework** to classify threats

### References
- [Application Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Log Analytics and KQL](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview)
- [Azure Sentinel documentation](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Logic Apps documentation](https://learn.microsoft.com/en-us/azure/logic-apps/)

---

## Week 4 — Deployment Strategies: Blue/Green (AKS) & Canary (ACA)

**Goal:** Deploy new versions of applications with zero downtime and the ability to instantly roll back.

### What you will build
- An AKS cluster with two deployments: `blue` (current) and `green` (new version)
- A Kubernetes Service that switches traffic from blue to green with one label change
- An Azure Container Apps environment with a Canary revision (10% → 50% → 100%)
- A CI/CD pipeline that automates the traffic shifting

### Visual — Blue/Green vs Canary

```
  Blue/Green (AKS):
  ┌──────────────────────────────────────────┐
  │                                          │
  │   Load Balancer / Ingress                │
  │         │                                │
  │    ┌────┴────┐                           │
  │    │         │                           │
  │  [BLUE]   [GREEN]                        │
  │  v1.0      v2.0                          │
  │  (live)   (ready)                        │
  │                                          │
  │  kubectl patch service → point to GREEN  │
  │  Instant switch. Roll back = point back. │
  └──────────────────────────────────────────┘

  Canary (ACA):
  ┌──────────────────────────────────────────┐
  │                                          │
  │   Ingress (traffic split rules)          │
  │         │                                │
  │   90%   │   10%                          │
  │    │         │                           │
  │  [v1.0]   [v2.0 canary]                  │
  │                                          │
  │  If metrics OK: 50% → 100%               │
  │  If metrics BAD: rollback to 0%          │
  └──────────────────────────────────────────┘
```

### Lab Steps — Blue/Green (AKS)
1. Create an AKS cluster (`az aks create`)
2. Deploy `blue` deployment with label `version: blue`
3. Create a Kubernetes Service selecting `version: blue`
4. Deploy `green` deployment with label `version: green`
5. Test green works: `kubectl port-forward`
6. Switch traffic: `kubectl patch service my-svc -p '{"spec":{"selector":{"version":"green"}}}'`
7. Verify traffic hit green, then roll back by switching selector back to blue

### Lab Steps — Canary (ACA)
1. Create an Azure Container Apps environment
2. Deploy v1 as the initial revision (100% traffic)
3. Deploy v2 as a new revision
4. Set traffic: `az containerapp ingress traffic set --revision-weight v1=90 v2=10`
5. Check Application Insights — compare error rates on v1 vs v2
6. Promote: set v2 to 50%, then 100%

### Key Concepts

| Term | Plain-English Meaning |
|------|-----------------------|
| Blue/Green | Two identical environments; switch all traffic instantly between them |
| Canary | Gradually shift a small % of traffic to new version; monitor; promote or roll back |
| Kubernetes Service | A stable DNS name + load balancer in front of pods |
| Label Selector | The mechanism Kubernetes uses to route traffic to the right pods |
| ACA Revision | An immutable snapshot of a Container App — you split traffic between revisions |
| Feature Flag | A code toggle that enables features for a subset of users (complements canary) |

### AZ-400 Exam Tips
- Know **when to use Blue/Green vs Canary** — Blue/Green is all-or-nothing; Canary is gradual
- Understand **rollback strategies** for both approaches
- ACA traffic splitting is done via `az containerapp ingress traffic set`
- AKS blue/green can also use **NGINX Ingress** or **Azure Application Gateway** instead of Service selectors

### References
- [AKS documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Azure Container Apps documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Blue/Green deployments on AKS (Microsoft)](https://learn.microsoft.com/en-us/azure/aks/blue-green-deployment)
- [ACA traffic splitting](https://learn.microsoft.com/en-us/azure/container-apps/traffic-splitting)
- [Helm documentation](https://helm.sh/docs/)

---

## Study Tools

| Tool | How to use |
|------|-----------|
| [Flashcards](../flashcards/app/index.html) | Open in browser — flip through all 5 modules daily |
| [Quizzes](../quizzes/quiz.html) | Take after each week — aim for 80%+ before moving on |
| [Microsoft Learn AZ-400 path](https://learn.microsoft.com/en-us/training/paths/az-400-develop-security-compliance-plan/) | Official free material — read alongside this repo |
| [Azure DevOps Labs](https://azuredevopslabs.com/) | Extra guided labs for pipeline practice |

---

## Similar Projects to Explore

| Project | What it adds |
|---------|-------------|
| [Azure/Enterprise-Scale (GitHub)](https://github.com/Azure/Enterprise-Scale) | Full ALZ Bicep reference implementation |
| [Azure DevOps Demo Generator](https://azuredevopsdemogenerator.azurewebsites.net/) | Pre-built Azure DevOps projects for practice |
| [johnthebrit/AZ-400 (GitHub)](https://github.com/johnthebrit/CertificationMaterials) | AZ-400 study notes and diagrams |
| [maddevsio/aws-eks-base](https://github.com/maddevsio/aws-eks-base) | See how another cloud does the same thing (context) |
| [DevSecOps-Studio (GitHub)](https://github.com/devsecops/devsecops-studio) | Docker-based DevSecOps lab environment |
