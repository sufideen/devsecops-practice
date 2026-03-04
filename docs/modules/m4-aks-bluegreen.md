---
layout: page
title: "Module 4: AKS Blue/Green Deployment"
permalink: /modules/m4-aks-bluegreen
---

# Module 4: AKS Blue/Green Deployment

> **Who this is for:** DevOps Engineers deploying containerised workloads, Solutions Architects designing resilient systems, AZ-400 candidates studying deployment strategies.

---

## The Big Picture — Why Blue/Green?

Blue/Green is a deployment strategy that eliminates downtime by maintaining **two identical production environments**. Only one serves live traffic at a time.

```
  Problem: Traditional deployment
  ────────────────────────────────────────────────────
  Deploy v2.0 → app goes down during update → users see errors
  If v2.0 breaks → you roll back → more downtime

  Solution: Blue/Green
  ────────────────────────────────────────────────────
  Blue (v1.0) is live. 100% traffic.
  Green (v2.0) is deployed. 0% traffic.
  Run smoke tests on Green.
  Switch traffic from Blue → Green. (Instant. Seconds.)
  If Green fails → switch back to Blue. (Instant rollback.)
  ────────────────────────────────────────────────────
  Result: Zero downtime. Instant rollback.
```

---

## Architecture Diagram

```
  Users / Internet
        │
        ▼
  Azure Load Balancer
  (or NGINX Ingress / App Gateway)
        │
        ▼
  ┌─────────────────────────────────────────────────────┐
  │                 AKS Cluster                         │
  │                                                     │
  │  Kubernetes Service: "my-app-svc"                   │
  │  selector: version: blue  ◄─── traffic goes here    │
  │        │                                            │
  │        ▼                                            │
  │  ┌────────────────┐    ┌────────────────┐           │
  │  │ BLUE Deployment│    │ GREEN Deployment│           │
  │  │ (v1.0)         │    │ (v2.0)         │           │
  │  │ replicas: 3    │    │ replicas: 3    │           │
  │  │ label:         │    │ label:         │           │
  │  │  version: blue │    │  version: green│           │
  │  └────────────────┘    └────────────────┘           │
  │                                                     │
  │  To switch: change selector from blue → green       │
  │  To rollback: change selector back to blue          │
  └─────────────────────────────────────────────────────┘
```

---

## Key Concepts Explained

### Kubernetes Deployments and Labels

```yaml
# Blue deployment — running v1.0
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue    # ← This label is the key
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: my-app
        image: myregistry.azurecr.io/my-app:v1.0
        ports:
        - containerPort: 8080
```

```yaml
# Green deployment — running v2.0
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green   # ← Different label
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: my-app
        image: myregistry.azurecr.io/my-app:v2.0
        ports:
        - containerPort: 8080
```

### Kubernetes Service — The Traffic Switch

```yaml
# Service — initially pointing to BLUE
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
    version: blue      # ← Change this to "green" to switch traffic
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

**To switch traffic from Blue to Green:**
```bash
kubectl patch service my-app-svc \
  -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'
```

**To roll back (Green → Blue):**
```bash
kubectl patch service my-app-svc \
  -p '{"spec":{"selector":{"app":"my-app","version":"blue"}}}'
```

### What Makes This "Instant"?

The Service change updates Kubernetes' iptables rules immediately. New connections go to Green pods within seconds. In-flight Blue requests complete normally. No downtime.

---

## Azure Container Registry (ACR)

Your Docker images are stored in ACR — Azure's private container registry.

```bash
# Build and push Blue (v1.0)
az acr build \
  --registry myregistry \
  --image my-app:v1.0 \
  --file Dockerfile .

# Build and push Green (v2.0) 
az acr build \
  --registry myregistry \
  --image my-app:v2.0 \
  --file Dockerfile .
```

AKS pulls from ACR using a **managed identity** — no credentials needed.

---

## Hands-On Lab

### Prerequisites
- Azure subscription
- Azure CLI + kubectl installed
- Docker installed (for local testing)

### Lab Steps

**Step 1 — Create an AKS cluster**
```bash
# Create resource group
az group create --name rg-aks-lab --location uksouth

# Create ACR
az acr create \
  --resource-group rg-aks-lab \
  --name myakslabregistry \
  --sku Basic

# Create AKS cluster attached to ACR
az aks create \
  --resource-group rg-aks-lab \
  --name aks-bluegreen-lab \
  --node-count 2 \
  --attach-acr myakslabregistry \
  --generate-ssh-keys

# Get credentials
az aks get-credentials \
  --resource-group rg-aks-lab \
  --name aks-bluegreen-lab
```

**Step 2 — Deploy Blue (v1.0)**
```bash
# Create a simple app for testing (or use your own)
cat > blue-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: my-app
        image: nginx:1.24    # Use nginx as a stand-in
        env:
        - name: VERSION
          value: "v1.0 BLUE"
        ports:
        - containerPort: 80
EOF

kubectl apply -f blue-deployment.yaml
```

**Step 3 — Create the Service (pointing to Blue)**
```bash
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
    version: blue
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f service.yaml

# Wait for external IP
kubectl get service my-app-svc --watch
```

**Step 4 — Deploy Green (v2.0) — without switching traffic**
```bash
cat > green-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: my-app
        image: nginx:1.25    # Newer version = Green
        env:
        - name: VERSION
          value: "v2.0 GREEN"
        ports:
        - containerPort: 80
EOF

kubectl apply -f green-deployment.yaml

# Verify green is healthy BEFORE switching
kubectl rollout status deployment/my-app-green
```

**Step 5 — Smoke test Green (before switching)**
```bash
# Test green directly using port-forward (no traffic switch yet)
kubectl port-forward deployment/my-app-green 8080:80

# In another terminal:
curl http://localhost:8080
# Should return GREEN response
```

**Step 6 — Switch traffic to Green**
```bash
kubectl patch service my-app-svc \
  -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'

# Verify
kubectl describe service my-app-svc | grep Selector
# Should show: version=green
```

**Step 7 — Rollback to Blue (if needed)**
```bash
kubectl patch service my-app-svc \
  -p '{"spec":{"selector":{"app":"my-app","version":"blue"}}}'
```

**Step 8 — Clean up**
```bash
az group delete --name rg-aks-lab --yes --no-wait
```

---

## Automating Blue/Green in a Pipeline

```yaml
# azure-pipelines.yml (Deploy stage)
- stage: Deploy_Green
  displayName: 'Deploy Green and Switch'
  jobs:
  - deployment: DeployGreen
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          # 1. Build and push green image
          - task: Docker@2
            inputs:
              containerRegistry: 'my-acr'
              repository: 'my-app'
              command: buildAndPush
              tags: 'green-$(Build.BuildId)'

          # 2. Deploy green deployment
          - task: KubernetesManifest@0
            inputs:
              action: deploy
              manifests: green-deployment.yaml

          # 3. Wait for green to be healthy
          - task: Kubernetes@1
            inputs:
              command: rollout
              arguments: status deployment/my-app-green

          # 4. Run smoke tests against green via port-forward
          - script: |
              kubectl port-forward deployment/my-app-green 8080:80 &
              sleep 5
              curl -f http://localhost:8080/health
            displayName: 'Smoke test green'

          # 5. Switch traffic (only if smoke tests passed)
          - task: Kubernetes@1
            inputs:
              command: patch
              arguments: >
                service my-app-svc
                -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'
            displayName: 'Switch traffic to Green'
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Switching traffic before green is healthy | Always run `kubectl rollout status` and smoke tests first |
| Deleting the blue deployment immediately after switch | Keep Blue alive for at least 24 hours for quick rollback |
| Not using readiness probes in the green deployment | Add `readinessProbe` to ensure pods are ready before traffic |
| Using the same image tag (e.g. `latest`) for blue and green | Always use immutable tags: `v1.0`, `v2.0`, or `$(Build.BuildId)` |

---

## AZ-400 Exam Tips

- Understand **when to use Blue/Green vs Rolling vs Canary** — Blue/Green is all-or-nothing, Rolling is gradual pod replacement, Canary is gradual traffic shift
- Know **Kubernetes readiness probes** vs **liveness probes** — readiness controls whether a pod receives traffic
- AKS supports **Azure Application Gateway Ingress Controller (AGIC)** for more sophisticated traffic routing
- **Helm** can simplify Blue/Green — package the deployment as a Helm chart with a `version` value

---

## Self-Check Questions

1. What is the key difference between Blue/Green and a Rolling deployment?
2. Which Kubernetes object controls which pods receive traffic from a Service?
3. Why should you NOT delete the Blue deployment immediately after switching to Green?
4. What command would you run to verify that the Green deployment is healthy before switching?
5. What is a readiness probe and why does it matter in Blue/Green?

> Answers are in the [Quiz](../../quizzes/quiz.html) — Module 4.

---

## References

- [AKS documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Blue/Green deployments on AKS (Microsoft)](https://learn.microsoft.com/en-us/azure/aks/blue-green-deployment)
- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Helm documentation](https://helm.sh/docs/)
- [NGINX Ingress for AKS](https://learn.microsoft.com/en-us/azure/aks/ingress-basic)
- [Application Gateway Ingress Controller](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)
