---
layout: page
title: "Module 3: Observability & Sentinel"
permalink: /modules/m3-observability
---

# Module 3: Observability & Sentinel

> **Who this is for:** Azure Admins managing production environments, Security Engineers building detection, AZ-400 candidates studying monitoring and instrumentation.

---

## The Big Picture — Why Observability Matters

You can only protect what you can see. Observability gives you visibility into what your application and infrastructure are doing — and alerts you when something goes wrong.

```
  Without observability:           With observability:
  ──────────────────────           ────────────────────
  Users complain →                 Alert fires at 99th percentile latency →
  You check logs manually →        Dashboard shows which service degraded →
  45 min to identify root cause    Root cause found in 5 min →
                                   Logic App auto-scales or notifies on-call
```

---

## Observability Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   Your Azure Environment                    │
  │                                                             │
  │  ┌──────────────┐    ┌────────────────┐    ┌─────────────┐ │
  │  │ Web App /    │    │ Azure Services  │    │ VMs / AKS   │ │
  │  │ Function App │    │ (Key Vault,     │    │ (agent or   │ │
  │  │              │    │  NSG, AFD...)   │    │  extension) │ │
  │  └──────┬───────┘    └───────┬─────────┘   └──────┬──────┘ │
  │         │                   │                      │        │
  │         ▼ SDK/agent         ▼ Diagnostic Settings  ▼ OMS    │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │           Application Insights                       │   │
  │  │  (requests, exceptions, dependencies, custom events) │   │
  │  └────────────────────┬─────────────────────────────────┘   │
  │                       │ linked workspace                     │
  │  ┌────────────────────▼─────────────────────────────────┐   │
  │  │           Log Analytics Workspace                    │   │
  │  │  (central store for ALL logs — query with KQL)       │   │
  │  └────────┬───────────────────────────┬─────────────────┘   │
  │           │                           │                      │
  │           ▼                           ▼                      │
  │  ┌────────────────┐        ┌──────────────────────────────┐ │
  │  │ Azure Monitor  │        │ Microsoft Sentinel            │ │
  │  │ Alert Rules    │        │ (SIEM — threat detection,     │ │
  │  │ Metric Alerts  │        │  SOAR — auto-remediation)     │ │
  │  └───────┬────────┘        └──────────────┬───────────────┘ │
  │          │                                │                  │
  │          ▼                                ▼                  │
  │  ┌────────────────┐        ┌──────────────────────────────┐ │
  │  │ Action Group   │        │ Logic App / Playbook          │ │
  │  │ (email, SMS,   │        │ (auto-notify Teams,           │ │
  │  │  webhook)      │        │  isolate VM, create ticket)   │ │
  │  └────────────────┘        └──────────────────────────────┘ │
  └─────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Explained

### Application Insights

Application Insights is Azure's **Application Performance Monitoring (APM)** tool. Add the SDK to your app and it automatically tracks:

| Data Type | What it captures |
|-----------|-----------------|
| Requests | Every HTTP call — URL, duration, status code |
| Exceptions | Every unhandled error with full stack trace |
| Dependencies | Calls to databases, APIs, queues |
| Custom Events | Business events you define (e.g., `OrderPlaced`) |
| Metrics | CPU, memory, request rates, failure rates |
| Live Metrics | Real-time view with 1-second latency |

**Distributed tracing:** Application Insights adds correlation IDs to requests so you can trace a single user journey across multiple microservices.

### Log Analytics & KQL

All logs flow into a **Log Analytics Workspace**. You query them with **KQL (Kusto Query Language)**.

**KQL quick examples:**

```kql
// Find all failed requests in the last hour
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by name, resultCode
| order by count_ desc

// Find exceptions by type
exceptions
| where timestamp > ago(24h)
| summarize count() by type
| order by count_ desc

// Find who deleted a resource (Activity Log)
AzureActivity
| where OperationNameValue == "Microsoft.Resources/subscriptions/resourceGroups/delete"
| project TimeGenerated, Caller, ResourceGroup, ActivityStatus
```

### Azure Monitor Alerts

Two main types:

```
  Metric Alerts                    Log Alerts
  ─────────────                    ───────────
  Based on real-time metrics       Based on KQL query results
  Near-instant firing              15-minute minimum frequency
  Example: CPU > 80% for 5 min    Example: error count > 10 in 1 hour
  Best for: infrastructure         Best for: application + security events
```

**Alert severity levels:**
- `Sev 0` — Critical (production down)
- `Sev 1` — Error (significant impact)
- `Sev 2` — Warning (degraded)
- `Sev 3` — Informational
- `Sev 4` — Verbose

### Microsoft Sentinel

Sentinel is Azure's **SIEM (Security Information and Event Management)** and **SOAR (Security Orchestration, Automation and Response)** platform.

```
  What Sentinel does:
  ───────────────────────────────────────────────
  1. Collect  → ingest logs from 200+ data connectors
  2. Detect   → analytics rules find suspicious patterns
  3. Investigate → incidents page, entity mapping, timeline
  4. Respond  → automation rules + playbooks (Logic Apps)
  ───────────────────────────────────────────────

  Example detection rule:
  "Alert if the same user logs in from two countries
   within 60 minutes" → Impossible Travel
```

**Data connectors include:**
- Azure Activity (who did what in Azure)
- Azure Active Directory (sign-in logs, audit logs)
- Microsoft Defender for Cloud
- Office 365
- Syslog from VMs
- 3rd party sources (Palo Alto, Cisco, etc.)

### Logic Apps — Automated Incident Response

When Sentinel raises an incident, a **Logic App playbook** can automatically:
- Post the incident details to a Teams channel
- Create a Jira/ServiceNow ticket
- Isolate the affected VM (block all traffic)
- Revoke a compromised user's sessions
- Send an email with remediation steps

---

## KQL Cheat Sheet (AZ-400 Must-Know)

```kql
// Basic structure
TableName
| where condition
| project column1, column2
| summarize count() by column1
| order by count_ desc
| take 10

// Time filters
| where TimeGenerated > ago(1h)
| where TimeGenerated between (datetime(2024-01-01) .. datetime(2024-01-31))

// String operations
| where Computer contains "prod"
| where Computer startswith "vm-"
| where Message matches regex "error.*timeout"

// Join two tables
TableA
| join kind=inner TableB on $left.RequestId == $right.RequestId

// Extending with new columns
| extend DurationSec = DurationMs / 1000
| extend IsError = iff(ResultCode >= 400, true, false)
```

---

## Hands-On Lab

### Prerequisites
- Azure subscription
- A sample web app (Azure App Service free tier works)
- Azure CLI installed

### Lab Steps

**Step 1 — Create a Log Analytics Workspace**
```bash
az monitor log-analytics workspace create \
  --resource-group rg-observability-lab \
  --workspace-name law-devsecops-lab \
  --location uksouth
```

**Step 2 — Create Application Insights linked to the workspace**
```bash
az monitor app-insights component create \
  --app appi-devsecops-lab \
  --location uksouth \
  --resource-group rg-observability-lab \
  --workspace /subscriptions/YOUR_SUB/resourceGroups/rg-observability-lab/providers/Microsoft.OperationalInsights/workspaces/law-devsecops-lab
```

**Step 3 — Enable diagnostic settings on Key Vault**
```bash
az monitor diagnostic-settings create \
  --name kv-to-law \
  --resource /subscriptions/YOUR_SUB/resourceGroups/rg-observability-lab/providers/Microsoft.KeyVault/vaults/YOUR_KV \
  --workspace /subscriptions/YOUR_SUB/resourceGroups/rg-observability-lab/providers/Microsoft.OperationalInsights/workspaces/law-devsecops-lab \
  --logs '[{"category":"AuditEvent","enabled":true}]'
```

**Step 4 — Create an alert rule**
```bash
az monitor metrics alert create \
  --name "high-error-rate" \
  --resource-group rg-observability-lab \
  --scopes /subscriptions/YOUR_SUB/resourceGroups/rg-observability-lab/providers/microsoft.insights/components/appi-devsecops-lab \
  --condition "avg requests/failed > 5" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Alert when failed requests exceed 5 in 5 minutes"
```

**Step 5 — Enable Microsoft Sentinel**
1. In Azure Portal → Microsoft Sentinel → Create → select your Log Analytics workspace
2. Go to Data connectors → connect "Azure Activity"
3. Go to Analytics → Rule templates → enable "Rare subscription-level operation in tenant"
4. Go to Automation → Create playbook (Logic App) that sends a Teams message on incident creation

**Step 6 — Test it**
- Generate some HTTP errors on your app
- Watch alerts fire in Azure Monitor
- Check the Sentinel Incidents page

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using classic Application Insights (not workspace-based) | Always create workspace-based App Insights — required for Sentinel integration |
| Creating alerts without action groups | Alerts with no action group fire silently — always attach an action group |
| Enabling Sentinel on a shared Log Analytics workspace | Use a dedicated workspace for Sentinel in production |
| Not setting data retention correctly | Default is 30 days — security compliance often requires 90 days minimum |

---

## AZ-400 Exam Tips

- Know the **three pillars of observability**: metrics, logs, traces
- KQL is tested — practice `summarize`, `join`, `extend`, `project`, `where`
- Understand **distributed tracing** with Application Insights — `operation_Id` links requests across services
- Know the difference between **metric alerts** (near real-time) and **log alerts** (query scheduled)
- **Deployment annotations** in App Insights — mark deployments so you can correlate releases with error spikes

---

## Self-Check Questions

1. What is the difference between Application Insights and Log Analytics?
2. Write a KQL query to find all failed HTTP requests in the last 24 hours.
3. What is the difference between Azure Monitor and Microsoft Sentinel?
4. What is a Logic App playbook in the context of Sentinel?
5. Why should you use a workspace-based Application Insights instead of classic?

> Answers are in the [Quiz](../../quizzes/quiz.html) — Module 3.

---

## References

- [Application Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview)
- [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Azure Monitor alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Microsoft Sentinel overview](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [Logic Apps documentation](https://learn.microsoft.com/en-us/azure/logic-apps/)
- [Sentinel playbooks (Logic Apps)](https://learn.microsoft.com/en-us/azure/sentinel/automate-responses-with-playbooks)
- [MITRE ATT&CK framework](https://attack.mitre.org/)
