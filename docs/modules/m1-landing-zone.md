---
layout: page
title: "Module 1: Azure Landing Zone (Bicep + Policy)"
permalink: /modules/m1-landing-zone
---

# Module 1: Azure Landing Zone (Bicep + Policy)

> **Who this is for:** Azure Admins setting up a new environment, Solutions Architects designing governance, AZ-400 candidates studying IaC and compliance.

---

## The Big Picture — What is a Landing Zone?

A **Landing Zone** is a pre-configured, secure Azure environment that workloads land into. Think of it like a well-built house: before you move furniture in, the electrical, plumbing, and security are all in place.

```
  WITHOUT a Landing Zone          WITH a Landing Zone
  ─────────────────────          ────────────────────
  Every team does their own      Central platform team sets up:
  networking, policies, and      ├── Networking (hub VNet, firewall)
  security from scratch.         ├── Identity (RBAC, PIM)
  Result: inconsistency,         ├── Governance (Azure Policy)
  security gaps, no auditability ├── Logging (Log Analytics)
                                 └── Security (Defender for Cloud)
                                 Teams deploy INTO this — consistently.
```

---

## Architecture Diagram

```
  Azure Active Directory Tenant
  │
  └── Root Management Group
      │
      ├── Platform (managed by central team)
      │   ├── Management Subscription
      │   │   ├── Log Analytics Workspace
      │   │   ├── Azure Monitor
      │   │   └── Microsoft Sentinel
      │   │
      │   ├── Connectivity Subscription
      │   │   ├── Hub Virtual Network
      │   │   ├── Azure Firewall
      │   │   ├── VPN / ExpressRoute Gateway
      │   │   └── Azure DNS Private Zones
      │   │
      │   └── Identity Subscription
      │       ├── Domain Controllers (AD DS)
      │       └── Azure AD Connect
      │
      └── Landing Zones (workload teams)
          ├── Corp Subscription (internal apps)
          │   ├── Spoke VNet (peered to Hub)
          │   └── [Your workloads here]
          │
          └── Online Subscription (internet-facing)
              ├── Spoke VNet (peered to Hub)
              └── [Your workloads here]
```

---

## Key Concepts Explained

### Management Groups

```
  Analogy: Management Groups are folders for subscriptions.
  Apply a policy at the folder level → it applies to ALL subscriptions inside.

  Root MG
  └── Company MG
      ├── Platform MG    ← Policies for platform team
      └── LandingZones MG
          ├── Corp MG    ← Policies for internal apps
          └── Online MG  ← Policies for internet-facing apps
```

### Azure Policy

Azure Policy enforces rules on resources automatically. You don't rely on people following a document — the platform enforces it.

| Effect | What it does |
|--------|-------------|
| `Deny` | Blocks resource creation if it violates the rule |
| `Audit` | Allows creation but flags it as non-compliant |
| `AuditIfNotExists` | Audits if a related resource is missing (e.g., no diagnostic setting) |
| `DeployIfNotExists` | Automatically deploys a companion resource if missing |
| `Modify` | Automatically adds/changes a tag or property |

**Example policy — require a tag:**
```json
{
  "if": {
    "field": "tags['environment']",
    "exists": "false"
  },
  "then": {
    "effect": "deny"
  }
}
```

### Bicep

Bicep is Microsoft's Infrastructure-as-Code language. It is the modern replacement for ARM JSON templates.

```bicep
// bicep/resource-group.bicep
targetScope = 'subscription'

param location string = 'uksouth'
param environment string

resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: 'rg-${environment}-app'
  location: location
  tags: {
    environment: environment
    managedBy: 'bicep'
  }
}
```

**Why Bicep over ARM JSON?**

| Feature | Bicep | ARM JSON |
|---------|-------|---------|
| Readability | Clean, concise | Verbose, nested |
| Modules | Yes — reuse `.bicep` files | Limited |
| Type safety | Yes | Limited |
| Azure Portal integration | Yes | Yes |
| Compiles to ARM | Yes (transparent) | Native |

### Conftest / OPA Policy Gates

Conftest lets you write policy rules in Rego that run **in your pipeline** before Bicep deploys.

```rego
# policies/required-tags.rego
package main

deny[msg] {
  input.resources[_].tags.environment == null
  msg := "All resources must have an 'environment' tag"
}
```

Run it: `conftest test bicep-output.json`

---

## Hands-On Lab

### Prerequisites
- Azure subscription (free tier works)
- Azure CLI installed: `az --version`
- Bicep CLI: `az bicep install`
- Conftest: download from [conftest.dev](https://www.conftest.dev/)

### Lab Steps

**Step 1 — Create a Management Group**
```bash
az account management-group create \
  --name "landing-zones" \
  --display-name "Landing Zones"
```

**Step 2 — Deploy a Bicep file**
```bash
# Create a simple Bicep file
cat > main.bicep << 'EOF'
targetScope = 'subscription'
param location string = 'uksouth'
param environment string = 'dev'

resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: 'rg-${environment}-lab'
  location: location
  tags: {
    environment: environment
    createdBy: 'lab'
  }
}
EOF

# Deploy it
az deployment sub create \
  --location uksouth \
  --template-file main.bicep \
  --parameters environment=dev
```

**Step 3 — Assign an Azure Policy**
```bash
# Assign built-in policy: require 'environment' tag
az policy assignment create \
  --name "require-env-tag" \
  --display-name "Require environment tag" \
  --policy "871b6d14-10aa-478d-b590-94f262ecfa99" \
  --scope "/subscriptions/YOUR_SUB_ID"
```

**Step 4 — Test your Conftest gate**
```bash
# Generate ARM template from Bicep
az bicep build --file main.bicep --outfile main.json

# Write a policy
mkdir policies
cat > policies/tags.rego << 'EOF'
package main
deny[msg] {
  r := input.resources[_]
  not r.tags.environment
  msg := sprintf("Resource %v is missing 'environment' tag", [r.name])
}
EOF

# Run Conftest
conftest test main.json
```

**Step 5 — Confirm it works**
- Remove the `environment` tag from your Bicep
- Run Conftest again — it should fail with your error message
- Add the tag back — Conftest passes
- This is your pipeline gate: if Conftest fails, the pipeline stops

---

## Common Mistakes (and How to Avoid Them)

| Mistake | Fix |
|---------|-----|
| Assigning policies at subscription level when they need to be at MG level | Use MG-level assignments for org-wide enforcement |
| Using `Deny` for everything | Use `Audit` first to see impact before blocking |
| Not testing Bicep locally before pipeline | Use `az bicep build` + `conftest test` locally first |
| Forgetting `targetScope` in Bicep | Set `targetScope = 'subscription'` for subscription-level deploys |

---

## AZ-400 Exam Tips

- Know that **Azure Policy initiatives** are groups of policy definitions — one assignment covers many rules
- Understand **policy compliance state** — `compliant`, `non-compliant`, `exempt`
- Bicep has a `what-if` operation: `az deployment sub what-if` — shows changes without deploying (used in pipelines)
- **Defender for Cloud** uses Azure Policy under the hood for its security recommendations

---

## Self-Check Questions

1. What is the difference between a Management Group and a Subscription?
2. Which Azure Policy effect would you use to automatically add a missing tag?
3. What does `conftest test` do in a pipeline context?
4. Why is Bicep preferred over ARM JSON for new projects?
5. What is the purpose of the Connectivity subscription in an ALZ design?

> Answers are in the [Quiz](../../quizzes/quiz.html) — Module 1.

---

## References

- [Azure Landing Zones overview](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Enterprise-Scale reference (GitHub)](https://github.com/Azure/Enterprise-Scale)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Policy documentation](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [OPA / Conftest](https://www.conftest.dev/)
- [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
- [Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction)
