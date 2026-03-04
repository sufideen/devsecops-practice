---
layout: default
title: Home
---

# DevSecOps Practice – AZ-400 Learning Hub

Welcome! This site is designed for **visual and kinaesthetic learners** — Azure Admins, Solutions Architects, newbie DevOps Engineers, and AZ-400 self-studiers.

---

## How to Use This Site

| If you want to... | Go to... |
|-------------------|---------|
| See the big picture and start quickly | [README](https://github.com/sufideen/devsecops-practice) |
| Follow a structured study plan | [Learning Path](./learning-path) |
| Deep-dive into a topic | [Modules](./modules/) |
| Memorise key terms | [Flashcards](../flashcards/app/index.html) |
| Test yourself AZ-400 style | [Quizzes](../quizzes/quiz.html) |

---

## Modules Overview

| Module | Topic | Key Tools |
|--------|-------|-----------|
| [M1](./modules/m1-landing-zone) | Azure Landing Zone (Bicep + Policy) | Bicep, Azure Policy, Conftest |
| [M2](./modules/m2-pipeline) | DevSecOps CI/CD Pipeline | CodeQL, Checkov, ZAP, TruffleHog |
| [M3](./modules/m3-observability) | Observability & Sentinel | App Insights, Log Analytics, Sentinel |
| [M4](./modules/m4-aks-bluegreen) | AKS Blue/Green Deployment | AKS, Helm, ACR |
| [M5](./modules/m5-aca-canary) | ACA Canary Deployment | Azure Container Apps, Dapr |

---

## AZ-400 Quick Reference

The AZ-400 exam tests your ability to **design and implement DevOps practices on Azure**.
The five modules in this repo map directly to the exam skill areas:

```
AZ-400 Skill Areas
├── Source Control            → M2 (branch strategy, PR gates)
├── Build & Release Pipelines → M2, M4, M5
├── Security & Compliance     → M1, M2
├── Instrumentation           → M3
└── Deployment Strategies     → M4, M5
```

> Official AZ-400 study guide: [Microsoft Learn – AZ-400](https://learn.microsoft.com/en-us/credentials/certifications/devops-engineer/)

---

> **Tip:** Do the lab in each module before reading theory. Kinaesthetic learners retain more by building first, then naming what they built.
