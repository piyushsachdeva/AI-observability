# Blame the Deploy

> Push bad code → ArgoCD deploys it → pod OOMKills → ask the AI agent → it tells you who broke it, what changed, and how to fix it.

---

## How It Works

```
git push bad commit
       │
       ├──► GitHub Actions → indexes commit to github-deployments (ES)
       │
       └──► ArgoCD (polls every 30s) → deploys to GKE cluster
                                              │
                                        paymentservice pod
                                              │
                                        OOMKilled (24Mi limit)
                                              │
                                        OTel DaemonSet → ships crash to logs-* (ES)
                                              │
                                   ┌──────────┴──────────┐
                                logs-*          github-deployments
                                   └──────────┬──────────┘
                                              │
                                        Agent Builder
                                              │
                              "commit a3f92b by @itsBaivab — that's the cause"
```

| Component | What it does |
|-----------|-------------|
| GKE Cluster | Runs 12 microservices + ArgoCD + OTel |
| Online Boutique | Google's demo e-commerce app — the thing we break |
| ArgoCD | Watches GitHub repo, auto-deploys every push |
| OTel kube-stack | Collects all pod logs + metrics → ships to Elastic Cloud |
| `github-deployments` | Custom ES index — stores who deployed what and when |
| `logs-*` | Pod logs in Elasticsearch — OOMKill events live here |
| Agent Builder | AI agent — correlates crash logs with deploy history |

**Why the order matters:**
1. Manifest must be pushed to GitHub **before** ArgoCD is created — ArgoCD deploys whatever is in the repo at first sync
2. Elastic Cloud must exist **before** OTel — OTel needs the endpoint + API key to send logs
3. `github-deployments` index must exist **before** GitHub secrets are added — first push writes to it
4. Everything must be running **before** the demo — agent needs data in both indices

---

## Prerequisites

```bash
# gcloud (macOS)
brew install --cask google-cloud-sdk
gcloud auth login
gcloud config set project <YOUR-GCP-PROJECT>

# gcloud (Linux)
curl https://sdk.cloud.google.com | bash && exec -l $SHELL
gcloud auth login
gcloud config set project <YOUR-GCP-PROJECT>

# kubectl
brew install kubectl                          # macOS
# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# helm
brew install helm                             # macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash  # Linux

# argocd CLI
brew install argocd                           # macOS
# Linux:
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# gh CLI
brew install gh && gh auth login              # macOS
sudo apt install gh -y && gh auth login       # Linux

# git
git config --global user.name "your-username"
git config --global user.email "your@email.com"
```

---

## Step 1 — GKE Cluster

```bash
# Create cluster
gcloud container clusters create elastic-cloud-test \
  --zone us-central1-a \
  --project <YOUR-GCP-PROJECT> \
  --no-enable-autoupgrade

# Add e2-standard-4 node pool
# (e2-standard-2 is too small — 12 services + OTel + ArgoCD need ~6 vCPU)
gcloud container node-pools create standard-pool \
  --cluster elastic-cloud-test \
  --zone us-central1-a \
  --project <YOUR-GCP-PROJECT> \
  --machine-type e2-standard-4 \
  --num-nodes 3

# Connect kubectl
gcloud container clusters get-credentials elastic-cloud-test \
  --zone us-central1-a \
  --project <YOUR-GCP-PROJECT>

# Verify
kubectl get nodes
# Expect: 3 nodes, Status: Ready
```

---

## Step 2 — Online Boutique Repo

The repo at `github.com/itsBaivab/microservices-demo` has been pre-configured:
- `release/kubernetes-manifests.yaml` — ArgoCD deploys this file
- `.github/workflows/index-deploy.yml` — indexes every push to Elasticsearch
- `strategy: type: Recreate` already added to 4 breakable services
- Istio manifests removed (we don't run Istio)

```bash
git clone https://github.com/itsBaivab/microservices-demo
cd microservices-demo
```

> **Do not edit anything yet.** Come back here in Step 8 (demo loop) to break things.

---

## Step 3 — Elastic Cloud

### 3a. Create deployment

1. `cloud.elastic.co` → **Create deployment**
2. Provider: **Google Cloud** | Region: **us-central1** | Name: `blame-the-deploy`
3. Click **Create** → **save the `elastic` password immediately** (shown once only)

### 3b. Get your endpoints

From `cloud.elastic.co` → click your deployment:

| What | Where to find it | Used for |
|------|-----------------|----------|
| Elasticsearch endpoint | Under **Elasticsearch** → **Copy endpoint** | GitHub secret `ES_ENDPOINT`, curl commands |
| OTLP ingest endpoint | Under **Integrations** → **Manage** → **APM** → copy the OTLP endpoint | OTel kube-stack secret `elastic_otlp_endpoint` |

### 3c. Create API key — open Kibana once

Kibana → ☰ → **Stack Management** → **Security** → **API Keys** → **Create API key**
- Name: `blame-the-deploy`
- Privileges: leave blank
- Click **Create** → **copy immediately** (shown once only)

Verify:
```bash
curl -s -w "\nHTTP:%{http_code}" \
  -X GET "<YOUR-ES-ENDPOINT>" \
  -H "Authorization: ApiKey <YOUR-API-KEY>"
# Expect: HTTP:200
```

---

## Step 4 — OpenTelemetry Kube-Stack

Ships all pod logs + metrics from GKE to Elastic Cloud automatically. Install once, never touch again.

```bash
# Add Helm repo
helm repo add open-telemetry 'https://open-telemetry.github.io/opentelemetry-helm-charts' --force-update

# Create namespace and secret
kubectl create namespace opentelemetry-operator-system

kubectl create secret generic elastic-secret-otel \
  --namespace opentelemetry-operator-system \
  --from-literal=elastic_otlp_endpoint='<YOUR-OTLP-INGEST-ENDPOINT>' \
  --from-literal=elastic_api_key='<YOUR-API-KEY>'

# Install
helm upgrade --install opentelemetry-kube-stack open-telemetry/opentelemetry-kube-stack \
  --namespace opentelemetry-operator-system \
  --values 'https://raw.githubusercontent.com/elastic/elastic-agent/refs/tags/v9.3.3/deploy/helm/edot-collector/kube-stack/managed_otlp/values.yaml' \
  --version '0.12.4'
```

> If Helm fails with conflict error: `helm uninstall opentelemetry-kube-stack -n opentelemetry-operator-system` then reinstall.

Verify:
```bash
kubectl get pods -n opentelemetry-operator-system
# Expect all Running:
# cluster-stats-collector   1/1 Running
# daemon-collector          1/1 Running  (×3, one per node)
# gateway-collector         1/1 Running  (×2)
# opentelemetry-operator    2/2 Running
```

---

## Step 5 — GitHub Deployments Index

Stores who deployed what and when — the bridge between crash logs and commit history.

### How the GitHub Actions workflow works

The repo already has `.github/workflows/index-deploy.yml`. On every push to `main` it:

1. **Detects which service changed** — diffs `release/kubernetes-manifests.yaml` between the last two commits, greps the added lines for a service name like `paymentservice`
2. **POSTs one JSON doc** to the `github-deployments` index in Elasticsearch:

```
timestamp   → when the push happened
commit_sha  → full git SHA
author      → GitHub username who pushed
service     → which service changed (detected from the diff)
change      → the commit message
diff_url    → GitHub compare URL showing exactly what changed
```

This is what the agent reads when you ask "who broke paymentservice?" — it gets back the author, SHA, and a link to the diff.

No Kubernetes component is involved — GitHub Actions runs in GitHub's cloud and POSTs directly to Elasticsearch.

### 5a. Create the index

```bash
curl -X PUT "<YOUR-ES-ENDPOINT>/github-deployments" \
  -H "Authorization: ApiKey <YOUR-API-KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "mappings": {
      "properties": {
        "timestamp":  { "type": "date" },
        "commit_sha": { "type": "keyword" },
        "author":     { "type": "keyword" },
        "service":    { "type": "keyword" },
        "image_tag":  { "type": "keyword" },
        "change":     { "type": "text" },
        "diff_url":   { "type": "keyword" }
      }
    }
  }'
# Expect: {"acknowledged":true}
```

### 5b. Add GitHub secrets

The workflow needs these to write to the index on every push:

```bash
gh secret set ES_ENDPOINT \
  --repo itsBaivab/microservices-demo \
  --body "<YOUR-ES-ENDPOINT>"

gh secret set ES_API_KEY \
  --repo itsBaivab/microservices-demo \
  --body "<YOUR-API-KEY>"
```

### 5c. Test it

```bash
git commit --allow-empty -m "test: verify ES indexing"
git push origin main

# Check it ran
gh run list --repo itsBaivab/microservices-demo --limit 3
# Expect: "Index deployment to Elasticsearch" → completed success

# Verify doc landed
curl -s "<YOUR-ES-ENDPOINT>/github-deployments/_count" \
  -H "Authorization: ApiKey <YOUR-API-KEY>"
# Expect: {"count":1,...}
```

---

## Step 6 — ArgoCD

Watches the repo and auto-deploys every push to the cluster.

```bash
# Install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s

# Expose publicly
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
# Note the EXTERNAL-IP

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo ""

# Log in
argocd login <EXTERNAL-IP> --username admin --password <PASSWORD> --insecure
```

### Create the Application

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: online-boutique
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/itsBaivab/microservices-demo
    targetRevision: main
    path: release
  destination:
    server: https://kubernetes.default.svc
    namespace: online-boutique
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### First sync + reduce poll interval

```bash
# Sync now
argocd app sync online-boutique --force --prune

# Reduce poll interval from 3 min to 30s
kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data": {"timeout.reconciliation": "30s"}}'
kubectl rollout restart deployment/argocd-repo-server -n argocd

# Verify all 12 pods running
kubectl get pods -n online-boutique
```

---

## Step 7 — Agent Builder

### 7a. Create LLM connector

Kibana → ☰ → **Stack Management** → **Connectors** → **Create connector** → **OpenAI**
- Name: `openai-blame`
- API key: your OpenAI key from `platform.openai.com`
- Model: `gpt-4o`
- Click **Save & test**

### 7b. Create the tools

Kibana → ☰ → **Search** → **Search AI Lake** → **Tools** → **Create tool**

**Tool 1 — `get_crash_logs`**
- Name: `get_crash_logs`
- Type: **Elasticsearch query**
- Index: `logs-*`
- ES|QL:

```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| WHERE body.text LIKE "*OOMKill*" OR body.text LIKE "*memory*"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, resource.attributes.k8s.deployment.name, resource.attributes.k8s.pod.name, body.text
```

Click **Save**. Then **Create tool** again for Tool 2.

**Tool 2 — `get_deploy_history`**
- Name: `get_deploy_history`
- Type: **Elasticsearch query**
- Index: `github-deployments`
- ES|QL:

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 5
| KEEP timestamp, author, commit_sha, service, change, diff_url
```

Click **Save**.

### 7c. Create the agent and add the tools

Kibana → ☰ → **Search** → **Search AI Lake** → **Agent Builder** → **Create agent**

- **Name:** `blame-the-deploy`
- **LLM connector:** `openai-blame`
- **Instructions:**

```
You are an SRE assistant. When asked why a service is crashing:
1. Use get_crash_logs to find OOMKill or memory events in logs-*
2. Use get_deploy_history to find the most recent GitHub deployment to that service
3. If the deploy happened shortly before the crash, that is the likely cause
4. Report: crash time, commit SHA, author, what changed, and how to fix it
Always cite the commit SHA and author name in your answer.
```

In the **Tools** section → **Add tool** → select `get_crash_logs` → **Add tool** again → select `get_deploy_history`

Click **Save**.

---

## Step 8 — Run the Demo

### Break it

Open `release/kubernetes-manifests.yaml`. Find paymentservice Deployment (~line 631). Change both memory values:

```yaml
resources:
  requests:
    memory: 24Mi   # was 64Mi
  limits:
    memory: 24Mi   # was 128Mi
```

```bash
git add release/kubernetes-manifests.yaml
git commit -m "perf: tune paymentservice memory limits for cost optimisation"
git push origin main
```

Watch:
```bash
kubectl get pods -n online-boutique -w
# paymentservice: Running → OOMKilled → CrashLoopBackOff
```

### Ask the agent

Kibana → ☰ → **Search** → **Search AI Lake** → **Agent Builder** → `blame-the-deploy`

```
Why is paymentservice crashing? Check the logs and recent deployments.
```

### Fix it

```bash
git revert HEAD --no-edit
git push origin main
# Pod recovers in ~30 seconds
```

```
Is paymentservice healthy now?
```

---

## Quick Reference

### Endpoints

| Service | URL |
|---------|-----|
| Online Boutique | `http://34.59.76.90` |
| ArgoCD | `https://136.116.115.80` → `admin` / `GoDWfO9NiMWzTD8W` |
| Kibana | `https://e177716345f94357bde1679d417f21ac.kb.us-central1.gcp.cloud.es.io` |
| Elasticsearch | `https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443` |

### Useful commands

```bash
kubectl get pods -n online-boutique          # check all pods
kubectl get pods -n online-boutique -w       # watch live
kubectl get pods -n opentelemetry-operator-system  # check OTel
argocd app get online-boutique               # ArgoCD sync status
argocd app sync online-boutique --force      # force sync
kubectl describe pod -n online-boutique <pod> | grep -A5 'Last State\|OOM'  # why did it crash
```

### ES|QL — find OOMKill events

```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| WHERE body.text LIKE "*OOMKill*"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, resource.attributes.k8s.deployment.name, body.text
```

### ES|QL — find recent deploys

```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 10
| KEEP timestamp, author, commit_sha, service, change
```
