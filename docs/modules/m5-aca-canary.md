---
layout: page
title: "Module 5: ACA Canary Deployment"
permalink: /modules/m5-aca-canary
---

# Module 5: Azure Container Apps — Canary Deployment

> **Who this is for:** DevOps Engineers managing containerised microservices, Solutions Architects designing low-risk release strategies, AZ-400 candidates studying deployment strategies.

---

## The Big Picture — What is a Canary Release?

A canary release sends a **small percentage of real user traffic** to a new version while the old version handles the rest. If the new version behaves well, you gradually increase its traffic share.

The name comes from the "canary in a coal mine" — miners sent canaries down first to detect danger before risking humans.

```
  Canary flow:
  ─────────────────────────────────────────────────────
  Step 1: Deploy v2.0 → send 10% of traffic to it
          Monitor: error rate, latency, user reports

  Step 2: Looks good → increase to 50%
          Monitor again

  Step 3: Stable → promote to 100%
          Decommission v1.0

  Step 4 (abort): v2.0 error rate spikes?
          → Set v2.0 to 0% → back to v1.0 instantly
  ─────────────────────────────────────────────────────
```

**Blue/Green vs Canary — at a glance:**

| | Blue/Green | Canary |
|--|-----------|--------|
| Traffic switch | All at once (0% or 100%) | Gradual (10% → 50% → 100%) |
| Risk | Low (instant rollback) | Very low (most users unaffected during test) |
| Rollback speed | Instant | Instant |
| Test with real traffic | No (smoke test only) | Yes |
| Complexity | Simple | Moderate |

---

## Azure Container Apps Architecture

```
  Users / Internet
        │
        ▼
  ACA Ingress (built-in load balancer + TLS)
        │
        │ Traffic split rules
        ├──── 90% ──────────────────────┐
        │                              │
        │                              ▼
        │                    ┌──────────────────────┐
        │                    │  Revision: v1.0       │
        │                    │  (stable, production) │
        │                    └──────────────────────┘
        │
        └──── 10% ──────────────────────┐
                                        │
                                        ▼
                              ┌──────────────────────┐
                              │  Revision: v2.0       │
                              │  (canary, new version)│
                              └──────────────────────┘

  Both revisions run in the SAME Container App environment.
  Traffic split is configured in the ingress rules.
```

---

## Key Concepts Explained

### Azure Container Apps vs AKS

| Feature | Azure Container Apps | AKS |
|---------|---------------------|-----|
| Kubernetes expertise needed | No (ACA manages it) | Yes |
| Managed Kubernetes | Fully managed | You manage node pools |
| Scaling | Built-in KEDA-based autoscaling | Manual or KEDA |
| Ingress | Built-in (no ingress controller to manage) | Requires ingress controller |
| Use case | Microservices, event-driven apps | Full container platform control |
| Pricing | Pay per use (CPU/memory seconds) | Pay per node (VM) |

### Revisions

In ACA, a **revision** is an immutable snapshot of your container app at a specific version. When you update your app, ACA creates a new revision. You split traffic between revisions.

```
  Container App: "my-app"
  ├── Revision: my-app--abc123  (v1.0 — deployed 2 weeks ago)
  │   └── Traffic: 90%
  └── Revision: my-app--def456  (v2.0 — deployed today)
      └── Traffic: 10%
```

### Traffic Splitting

```bash
# Set traffic: 90% to v1.0 revision, 10% to v2.0 canary
az containerapp ingress traffic set \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision-weight \
    my-app--abc123=90 \
    my-app--def456=10
```

---

## Hands-On Lab

### Prerequisites
- Azure subscription
- Azure CLI with Container Apps extension
- Docker installed

### Lab Steps

**Step 1 — Install ACA extension and register providers**
```bash
az extension add --name containerapp --upgrade

az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

**Step 2 — Create environment**
```bash
az group create --name rg-aca-lab --location uksouth

# Create Log Analytics workspace for ACA
az monitor log-analytics workspace create \
  --resource-group rg-aca-lab \
  --workspace-name law-aca-lab

LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-aca-lab \
  --workspace-name law-aca-lab \
  --query customerId -o tsv)

LAW_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group rg-aca-lab \
  --workspace-name law-aca-lab \
  --query primarySharedKey -o tsv)

# Create ACA environment
az containerapp env create \
  --name aca-env-lab \
  --resource-group rg-aca-lab \
  --location uksouth \
  --logs-workspace-id "$LAW_ID" \
  --logs-workspace-key "$LAW_KEY"
```

**Step 3 — Deploy v1.0 (initial revision)**
```bash
az containerapp create \
  --name my-app \
  --resource-group rg-aca-lab \
  --environment aca-env-lab \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3 \
  --revision-suffix v1

# Get the URL
az containerapp show \
  --name my-app \
  --resource-group rg-aca-lab \
  --query properties.configuration.ingress.fqdn -o tsv
```

**Step 4 — Enable multiple active revisions**
```bash
az containerapp revision set-mode \
  --name my-app \
  --resource-group rg-aca-lab \
  --mode multiple
```

**Step 5 — Deploy v2.0 (canary revision)**
```bash
# Update the image tag to simulate a new version
az containerapp update \
  --name my-app \
  --resource-group rg-aca-lab \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --revision-suffix v2 \
  --set-env-vars "VERSION=v2.0"
```

**Step 6 — List revisions**
```bash
az containerapp revision list \
  --name my-app \
  --resource-group rg-aca-lab \
  --output table
# Note the revision names for the next step
```

**Step 7 — Set 10% canary traffic to v2**
```bash
# Replace revision names with your actual revision names from Step 6
V1_REVISION="my-app--v1"
V2_REVISION="my-app--v2"

az containerapp ingress traffic set \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision-weight \
    ${V1_REVISION}=90 \
    ${V2_REVISION}=10
```

**Step 8 — Monitor v2 in Application Insights**
```bash
# Check if v2 has any errors
az containerapp logs show \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision $V2_REVISION \
  --follow
```

**Step 9 — Promote to 50% → 100%**
```bash
# Promote to 50%
az containerapp ingress traffic set \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision-weight ${V1_REVISION}=50 ${V2_REVISION}=50

# Monitor for 5-10 minutes. If stable:

# Promote to 100%
az containerapp ingress traffic set \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision-weight ${V1_REVISION}=0 ${V2_REVISION}=100

# Deactivate v1
az containerapp revision deactivate \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision $V1_REVISION
```

**Step 10 — Emergency rollback**
```bash
# If v2 is causing errors — roll back instantly
az containerapp ingress traffic set \
  --name my-app \
  --resource-group rg-aca-lab \
  --revision-weight ${V1_REVISION}=100 ${V2_REVISION}=0
```

**Cleanup**
```bash
az group delete --name rg-aca-lab --yes --no-wait
```

---

## KEDA Autoscaling in ACA

Azure Container Apps uses **KEDA (Kubernetes Event-Driven Autoscaling)** to scale based on events — not just CPU.

```yaml
# Scale on HTTP request queue depth
scaleRules:
  - name: http-scaler
    http:
      metadata:
        concurrentRequests: 10   # Scale up when >10 concurrent requests per replica

# Scale on Azure Service Bus queue length
scaleRules:
  - name: sb-scaler
    custom:
      type: azure-servicebus
      metadata:
        queueName: orders
        messageCount: "100"      # Scale up when >100 messages in queue
```

---

## Automating Canary in a Pipeline

```yaml
# azure-pipelines.yml
- stage: Canary_10
  displayName: 'Canary 10%'
  jobs:
  - deployment: Canary10
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          # Deploy new revision
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'my-service-connection'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az containerapp update \
                  --name my-app \
                  --resource-group rg-prod \
                  --image myregistry.azurecr.io/my-app:$(Build.BuildId) \
                  --revision-suffix "$(Build.BuildId)"

                # Set 10% traffic to new revision
                NEW_REV=$(az containerapp revision list \
                  --name my-app \
                  --resource-group rg-prod \
                  --query "[0].name" -o tsv)

                OLD_REV=$(az containerapp revision list \
                  --name my-app \
                  --resource-group rg-prod \
                  --query "[1].name" -o tsv)

                az containerapp ingress traffic set \
                  --name my-app \
                  --resource-group rg-prod \
                  --revision-weight ${OLD_REV}=90 ${NEW_REV}=10

          # Wait and check metrics (or use a manual approval gate here)
          - task: ManualValidation@0
            inputs:
              instructions: 'Check Application Insights for the new revision. If error rate is acceptable, approve to promote to 50%.'
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting to enable multiple revision mode | Run `az containerapp revision set-mode --mode multiple` first |
| Not monitoring the canary revision in App Insights | Filter by revision name in Application Insights queries |
| Sending too much traffic to canary too fast | Start with 5-10% and wait at least 15 minutes per step |
| Using `:latest` tag for revisions | Use immutable tags like `v2.0` or `$(Build.BuildId)` |
| Not deactivating old revisions | Old inactive revisions consume resources — deactivate when no longer needed |

---

## AZ-400 Exam Tips

- Know that ACA traffic splitting is done via **ingress traffic weight** — not Kubernetes Services
- Understand **revision lifecycle**: active, inactive, deprovisioned
- ACA integrates with **Dapr** for microservice communication patterns
- Know the difference between **ACA environments** (shared network/logging boundary) and **container apps** (individual apps within an environment)
- **KEDA** is the autoscaling engine — know common scalers: HTTP, Azure Service Bus, Azure Storage Queue

---

## Self-Check Questions

1. What is the difference between a Canary deployment and a Blue/Green deployment?
2. What is an ACA revision?
3. What command sets traffic to 10% on a new revision and 90% on the old?
4. What is KEDA and what problem does it solve?
5. Why should you use immutable image tags instead of `:latest`?

> Answers are in the [Quiz](../../quizzes/quiz.html) — Module 5.

---

## References

- [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [ACA traffic splitting](https://learn.microsoft.com/en-us/azure/container-apps/traffic-splitting)
- [ACA revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [KEDA documentation](https://keda.sh/docs/)
- [Dapr documentation](https://dapr.io/docs/)
- [ACA managed identities](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity)
- [ACA and Azure DevOps pipelines](https://learn.microsoft.com/en-us/azure/container-apps/github-actions)
