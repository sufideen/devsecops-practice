---
layout: page
title: "Module 2: DevSecOps CI/CD Pipeline"
permalink: /modules/m2-pipeline
---

# Module 2: DevSecOps CI/CD Pipeline

> **Who this is for:** DevOps Engineers building pipelines, Security Engineers embedding security checks, AZ-400 candidates studying pipeline security.

---

## The Big Picture — Shift-Left Security

"Shift-left" means moving security checks earlier in the development lifecycle. Finding a vulnerability in source code costs far less to fix than finding it in production.

```
  Cost to fix a vulnerability:
  ─────────────────────────────────────────────────────
  Design  │ $
  Code    │ $$
  Build   │ $$$
  Test    │ $$$$
  Deploy  │ $$$$$
  Prod    │ $$$$$$$$$$  ← We want to avoid this!
  ─────────────────────────────────────────────────────

  Shift-Left: catch issues at Code and Build — not Prod.
```

---

## Pipeline Architecture

```
  Developer pushes code
          │
          ▼
  ┌───────────────────────────────────────────────────────┐
  │  Stage 1: Build & SAST                                │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  Checkout code                                   │ │
  │  │  ├── CodeQL scan     (finds code vulnerabilities)│ │
  │  │  ├── Semgrep scan    (custom security rules)     │ │
  │  │  ├── TruffleHog scan (finds secrets in commits)  │ │
  │  │  └── Build artifact (Docker image / app package) │ │
  │  └──────────────────────────────────────────────────┘ │
  │              │ PASS? Continue │ FAIL? Stop pipeline    │
  ├───────────────────────────────────────────────────────┤
  │  Stage 2: IaC Security Scan                           │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  ├── Checkov (Bicep/Terraform misconfig check)   │ │
  │  │  └── Conftest/OPA (custom policy rules)          │ │
  │  └──────────────────────────────────────────────────┘ │
  │              │ PASS? Continue │ FAIL? Stop pipeline    │
  ├───────────────────────────────────────────────────────┤
  │  Stage 3: Deploy to Test Environment                  │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  Deploy app to a short-lived test environment    │ │
  │  └──────────────────────────────────────────────────┘ │
  ├───────────────────────────────────────────────────────┤
  │  Stage 4: DAST                                        │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  OWASP ZAP baseline scan against test app        │ │
  │  └──────────────────────────────────────────────────┘ │
  │              │ PASS? Continue │ FAIL? Stop pipeline    │
  ├───────────────────────────────────────────────────────┤
  │  Stage 5: Deploy to Production (Manual Approval Gate) │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │  Human approves → Bicep/Helm deploy to Azure     │ │
  │  └──────────────────────────────────────────────────┘ │
  └───────────────────────────────────────────────────────┘
```

---

## Security Tools Explained

### SAST — Static Application Security Testing

SAST reads your source code **without running it** and looks for patterns that indicate vulnerabilities.

```
  Your code:
  ─────────────────────────────────────
  string query = "SELECT * FROM users
                  WHERE id = " + userId;
  ─────────────────────────────────────
  SAST flags this as SQL Injection risk!
  userId could be: "1 OR 1=1; DROP TABLE users--"
```

| Tool | Best for | Language support |
|------|---------|-----------------|
| CodeQL | Complex vulnerability patterns | C, C++, C#, Java, JavaScript, Python, Ruby, Go |
| Semgrep | Fast, custom rules, CI-friendly | 30+ languages |
| Bandit | Python-specific | Python only |

**AZ-400 note:** GitHub Advanced Security includes CodeQL — available in Azure DevOps via the GitHub Advanced Security for Azure DevOps extension.

### DAST — Dynamic Application Security Testing

DAST attacks your **running application** — it sends malicious requests and looks for vulnerable responses.

```
  OWASP ZAP attacks your app like a hacker would:
  ─────────────────────────────────────────────────
  ├── Tries SQL injection in form fields
  ├── Tries XSS in URL parameters
  ├── Checks for missing security headers
  ├── Looks for exposed admin interfaces
  └── Tests CORS configuration
```

**Baseline scan** = passive only (safe for CI/CD)
**Full scan** = active attack (use in dedicated test environment only)

### Secrets Scanning — TruffleHog

TruffleHog scans your **entire git history** for secrets that were ever committed — even deleted ones.

```bash
# What TruffleHog looks for:
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"     ← AWS key
AZURE_CLIENT_SECRET = "abc123..."               ← Azure service principal secret
password = "SuperSecret123"                     ← Hardcoded password
Authorization: Bearer eyJhbGci...               ← JWT token
```

**Rule:** Never commit secrets. Use Azure Key Vault + managed identities instead.

### IaC Scanning — Checkov

Checkov scans Bicep and Terraform for misconfigurations **before** they reach Azure.

Common findings Checkov catches:

| Finding | Risk |
|---------|------|
| Storage account allows public blob access | Data exposure |
| NSG allows inbound 0.0.0.0/0 on port 22 | SSH open to internet |
| Key Vault soft delete disabled | Accidental secret deletion |
| No diagnostic settings on resources | No audit trail |
| AKS without RBAC enabled | Unauthorized cluster access |

---

## Sample Pipeline YAML (Azure DevOps)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main, develop]

stages:

- stage: Build_SAST
  displayName: 'Build & SAST Scan'
  jobs:
  - job: SAST
    pool:
      vmImage: ubuntu-latest
    steps:
    - checkout: self
      fetchDepth: 0   # TruffleHog needs full history

    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'

    # TruffleHog secrets scan
    - script: |
        pip install trufflehog
        trufflehog git file://. --since-commit HEAD~1 --fail
      displayName: 'TruffleHog Secrets Scan'

    # Semgrep SAST
    - script: |
        pip install semgrep
        semgrep --config=auto --error .
      displayName: 'Semgrep SAST Scan'

    - task: Docker@2
      displayName: 'Build Docker Image'
      inputs:
        command: build
        dockerfile: Dockerfile
        tags: $(Build.BuildId)

- stage: IaC_Scan
  displayName: 'IaC Security Scan'
  dependsOn: Build_SAST
  jobs:
  - job: Checkov
    pool:
      vmImage: ubuntu-latest
    steps:
    - script: |
        pip install checkov
        checkov -d ./infrastructure --framework bicep --soft-fail false
      displayName: 'Checkov IaC Scan'

- stage: Deploy_Test
  displayName: 'Deploy to Test'
  dependsOn: IaC_Scan
  jobs:
  - deployment: DeployTest
    environment: 'test'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo "Deploy to test environment"

- stage: DAST
  displayName: 'DAST Scan'
  dependsOn: Deploy_Test
  jobs:
  - job: ZAP
    pool:
      vmImage: ubuntu-latest
    steps:
    - script: |
        docker run -t owasp/zap2docker-stable zap-baseline.py \
          -t https://your-test-app.azurewebsites.net \
          -r zap-report.html
      displayName: 'OWASP ZAP Baseline Scan'

- stage: Deploy_Prod
  displayName: 'Deploy to Production'
  dependsOn: DAST
  jobs:
  - deployment: DeployProd
    environment: 'production'   # Has approval gate configured
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'my-service-connection'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az deployment group create \
                  --resource-group rg-prod \
                  --template-file infrastructure/main.bicep
```

---

## Branch Strategy for DevSecOps

```
  main ────────────────────────────────► (protected, production)
    ↑                                      requires: PR + 2 approvals
    │                                                + all checks pass
  develop ─────────────────────────────► (integration branch)
    ↑
  feature/add-login ──────────────────► (short-lived, one feature)
    ↑
  You work here → PR to develop → merge → PR to main → release
```

**Branch protection rules to set:**
- Require pull request reviews (minimum 1-2 approvers)
- Require status checks to pass (SAST, IaC scan)
- Require branches to be up to date before merging
- Restrict who can push to `main`

---

## SBOM & SCA — Supply Chain Security

```
  Your app uses 150 npm packages.
  Each of those uses more packages.
  Some of those packages have known CVEs.

  SBOM = Software Bill of Materials
         (a machine-readable list of everything your app uses)

  SCA  = Software Composition Analysis
         (a tool that checks your SBOM against CVE databases)

  Tools: Syft (SBOM generation), Grype (vulnerability scan), Dependabot
```

---

## Hands-On Lab

### Prerequisites
- Azure DevOps organisation (free tier at dev.azure.com)
- Docker installed
- Python 3.x for running scanners locally

### Lab Steps

1. **Fork this repo** and create an Azure DevOps pipeline linked to it
2. **Add TruffleHog** — run it locally first: `trufflehog git file://. --since-commit HEAD~1`
3. **Deliberately add a fake secret** to a test branch: `export FAKE_KEY="AKIAIOSFODNN7EXAMPLE"` in a script file
4. **Run TruffleHog again** — confirm it catches the fake key
5. **Add Checkov** — run `checkov -d . --framework bicep` on the bicep files from Module 1
6. **Introduce a misconfiguration** — remove `enableSoftDelete: true` from a Key Vault Bicep file
7. **Run Checkov** — confirm it flags CKV_AZURE_110
8. **Add all steps to a pipeline YAML** and trigger it with a push

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running DAST against production | Always use a dedicated test environment for DAST |
| Using `--soft-fail true` on Checkov in main branch pipeline | Set `--soft-fail false` for main — gates must block |
| Not scanning git history (only HEAD) | TruffleHog needs `fetchDepth: 0` and `--since-commit` flag |
| Storing secrets in pipeline variables (plain text) | Use Azure Key Vault + variable groups with secret masking |

---

## AZ-400 Exam Tips

- Know the difference between **service connections** (Azure DevOps) and **service principals** (Azure AD)
- Understand **environment approvals** in Azure DevOps — how to require a human sign-off before deploying to prod
- **GitHub Advanced Security for Azure DevOps** brings CodeQL scanning natively into Azure Pipelines
- Know what a **dependency track** is in SCA context
- **OWASP Top 10** — SAST/DAST tools map their findings to OWASP categories

---

## Self-Check Questions

1. What is the difference between SAST and DAST? Give an example of each.
2. Why must TruffleHog scan the full git history, not just the latest commit?
3. What does Checkov check, and what happens when it finds a HIGH severity finding?
4. At which stage of the pipeline would you run OWASP ZAP?
5. What is an SBOM and why does it matter for supply chain security?

> Answers are in the [Quiz](../../quizzes/quiz.html) — Module 2.

---

## References

- [DevSecOps on Azure (Microsoft)](https://learn.microsoft.com/en-us/azure/devops/devsecops/)
- [GitHub Advanced Security for Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features)
- [CodeQL documentation](https://codeql.github.com/docs/)
- [Semgrep documentation](https://semgrep.dev/docs/)
- [TruffleHog (GitHub)](https://github.com/trufflesecurity/trufflehog)
- [Checkov IaC scanner (GitHub)](https://github.com/bridgecrewio/checkov)
- [OWASP ZAP](https://www.zaproxy.org/)
- [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Azure DevOps pipeline YAML schema](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/)
- [Syft SBOM generator](https://github.com/anchore/syft)
