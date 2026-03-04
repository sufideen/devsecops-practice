# DevSecOps Practice – AZ-400 Study & Hands-On Lab Repo

> **Who is this for?**
> Azure Admins, Solutions Architects, newbie Azure DevOps Engineers, and AZ-400 self-learners who prefer learning by **doing**, **seeing**, and **building** — not just reading slides.

---

## What is DevSecOps? (30-second picture)

```
  ┌──────────────────────────────────────────────────────────────┐
  │                     DevSecOps Loop                           │
  │                                                              │
  │  Plan → Code → Build → Test → Release → Deploy → Monitor    │
  │    ↑       ↑      ↑       ↑       ↑        ↑        ↑       │
  │  Threat  SAST  Policy  DAST   Approval  IaC     Sentinel     │
  │  Model   Scan   Gate   Scan    Gate     Scan    Alerting     │
  └──────────────────────────────────────────────────────────────┘
```

Security is **not a gate at the end** — it is woven into every stage. This repo gives you hands-on practice with each stage using real Azure tooling.

---

## Repository Map

```
devsecops-practice/
├── docs/                        ← Guided module pages (GitHub Pages)
│   ├── index.md
│   ├── learning-path.md
│   └── modules/
│       ├── m1-landing-zone.md   ← Module 1: Azure Landing Zone
│       ├── m2-pipeline.md       ← Module 2: DevSecOps CI/CD Pipeline
│       ├── m3-observability.md  ← Module 3: Observability & Sentinel
│       ├── m4-aks-bluegreen.md  ← Module 4: AKS Blue/Green Deploy
│       └── m5-aca-canary.md     ← Module 5: ACA Canary Deploy
├── flashcards/app/index.html    ← Interactive flip-card study tool
├── quizzes/quiz.html            ← AZ-400-style self-assessment
├── proof/                       ← Paste your screenshots here
├── SECURITY.md
└── README_PORTFOLIO.md          ← Portfolio / CV evidence page
```

---

## 5 Modules at a Glance

| # | Module | Core Azure Services | AZ-400 Domain |
|---|--------|---------------------|---------------|
| 1 | Azure Landing Zone (Bicep + Policy) | Management Groups, Bicep, Azure Policy, Defender for Cloud | Design & Implement Pipelines |
| 2 | DevSecOps CI/CD Pipeline | Azure DevOps, GitHub Actions, CodeQL, Checkov, OWASP ZAP | Security in Pipelines |
| 3 | Observability & Sentinel | App Insights, Log Analytics, Azure Sentinel, Logic Apps | Monitoring & Feedback |
| 4 | AKS Blue/Green Deployment | AKS, Helm, Azure Load Balancer, Container Registry | Deploy Apps |
| 5 | ACA Canary Deployment | Azure Container Apps, Traffic Splitting, Dapr | Deploy Apps |

---

## Start Here — Quickstart in 4 Steps

### Step 1 — Fork & clone

```bash
git clone https://github.com/sufideen/devsecops-practice.git
cd devsecops-practice
```

### Step 2 — Pick your starting point

| Your background | Start with |
|-----------------|------------|
| Azure Admin (VMs, networking, subscriptions) | Module 1 → Module 2 |
| Solutions Architect (design, security, governance) | Module 1 → Module 3 |
| Newbie DevOps Engineer (CI/CD first) | Module 2 → Module 4 |
| Pure AZ-400 exam prep | Flashcards → Quizzes → Modules |

### Step 3 — Use the interactive tools

- **Flashcards:** `flashcards/app/index.html` — open in browser, flip cards per module
- **Quizzes:** `quizzes/quiz.html` — AZ-400-style MCQs with scored answers

### Step 4 — Do the labs

Each module has:
1. A visual architecture diagram (ASCII + description)
2. Step-by-step lab instructions
3. Key concepts explained in plain English
4. AZ-400 exam tips
5. References to official Microsoft Learn paths

---

## AZ-400 Exam Domain Coverage

| AZ-400 Domain | Covered In |
|---------------|-----------|
| Design and implement processes and communications | M1, M2 |
| Design and implement a source control strategy | M2 |
| Design and implement build and release pipelines | M2, M4, M5 |
| Develop a security and compliance plan | M1, M2 |
| Implement an instrumentation strategy | M3 |
| Design and implement deployment strategies | M4, M5 |

> **Official exam page:** [AZ-400 Microsoft Azure DevOps Engineer Expert](https://learn.microsoft.com/en-us/credentials/certifications/devops-engineer/)

---

## Similar Projects & Further Reading

| Resource | Why useful |
|----------|-----------|
| [Azure Landing Zones (Microsoft)](https://github.com/Azure/Enterprise-Scale) | Reference architecture this repo is based on |
| [Azure DevOps Labs](https://azuredevopslabs.com/) | Free hands-on labs with guided walkthroughs |
| [AZ-400 Microsoft Learn Path](https://learn.microsoft.com/en-us/training/paths/az-400-develop-security-compliance-plan/) | Official free study material |
| [DevSecOps on Azure (MS Docs)](https://learn.microsoft.com/en-us/azure/devops/devsecops/) | Conceptual reference |
| [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/) | Security testing standards used in M2 |
| [Checkov IaC Scanner](https://github.com/bridgecrewio/checkov) | Policy-as-code tool used in M2 |
| [OpenSSF Scorecard](https://github.com/ossf/scorecard) | Supply chain security scoring |
| [TechWorld with Nana – DevOps Bootcamp](https://www.youtube.com/@TechWorldwithNana) | Visual video learning companion |
| [Microsoft Reactor DevSecOps Series](https://developer.microsoft.com/en-us/reactor/) | Live sessions and recordings |

---

## Proof of Work

Add screenshots of your pipeline runs, policy violations caught, Sentinel incidents, and deployment logs to the `/proof/` folder. This builds your portfolio evidence for job applications.

---

## License

MIT — free to use, adapt, and share.
