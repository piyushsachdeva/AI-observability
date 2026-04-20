# Blame the Deploy

> **One-line pitch:** Push bad code → ArgoCD deploys it → pod OOMKills → ask the AI agent → it tells you who broke it, what they changed, and how to fix it.

---

## How It Works — Full Architecture

```
Developer pushes bad commit
        │
        ▼
GitHub (itsBaivab/microservices-demo)
        │
        ├──► GitHub Actions (index-deploy.yml)
        │         │
        │         └──► Elasticsearch: github-deployments index
        │                  stores: commit SHA, author, service, what changed
        │
        └──► Webhook ──► ArgoCD (GitOps controller)
                              │
                              └──► GKE Cluster (elastic-cloud-test)
                                        │
                                        └──► online-boutique namespace
                                                  │
                                                  └──► paymentservice pod
                                                            │
                                                            └──► OOMKilled (24Mi limit)
                                                                      │
                                                                      ▼
                                              OpenTelemetry kube-stack (DaemonSet on every node)
                                                                      │
                                                                      └──► Elastic Cloud
                                                                                │
                                                              ┌─────────────────┤
                                                              │                 │
                                                         logs-*          github-deployments
                                                              │                 │
                                                              └────────┬────────┘
                                                                       │
                                                                  Agent Builder
                                                                       │
                                                                       ▼
                                              "paymentservice OOMKilled at 14:32.
                                               Last deploy: commit a3f92b by @itsBaivab at 14:28.
                                               Changed: memory 128Mi → 24Mi.
                                               Fix: git revert a3f92b && git push"
```

### What each component does

| Component | Role |
|-----------|------|
| **GKE Cluster** | Runs 12 microservices (Online Boutique) + all tooling |
| **Online Boutique** | Google's demo e-commerce app — the thing we break |
| **ArgoCD** | GitOps controller — watches the GitHub repo, deploys every push automatically |
| **GitHub Webhook** | Tells ArgoCD about new commits instantly (no 3-min poll wait) |
| **GitHub Actions** | On every push, indexes commit metadata to Elasticsearch |
| **`github-deployments` index** | Custom ES index — who deployed what, when, to which service |
| **OTel kube-stack** | DaemonSet on every node — ships all pod logs + metrics to Elastic Cloud |
| **`logs-*` index** | Pod logs in Elasticsearch — includes OOMKill events, crash traces |
| **Agent Builder** | AI agent with two ES tools — correlates crash logs with deploy history |

### The key insight

Elasticsearch alone can tell you *a pod crashed*. GitHub alone can tell you *a commit was pushed*. Neither can tell you the full story on its own.

`github-deployments` is the bridge. Every push writes a record: timestamp, author, service, commit SHA, what changed. When the agent sees an OOMKill at 14:32 and a deploy to the same service at 14:28, it connects the dots.

### Why Recreate strategy?

Paymentservice, cartservice, frontend, and recommendationservice all use `strategy: type: Recreate`. This means when a new deployment rolls out, Kubernetes **kills the old pod first** before starting the new one. There's a real outage — the crash is visible immediately. Without this, a rolling update would keep the old pod alive while the new one starts, masking the problem.

### What ArgoCD does during the demo — and what it does NOT do

**What ArgoCD DOES:**
- Detects the new commit within seconds (via GitHub webhook)
- Applies the updated manifest to the cluster — deploys the bad pod with 24Mi memory
- Shows the sync as **Synced** (green) — because it successfully applied what was in the repo

**What ArgoCD does NOT do:**
- Does NOT detect that the pod is crashing and revert the change
- Does NOT heal or roll back — `selfHeal: false` means it only syncs repo → cluster, not cluster → repo
- Does NOT know or care that 24Mi is too small — it just applies the manifest

**Why the pod keeps restarting (not ArgoCD's fault):**
- Kubernetes sees the pod crash (OOMKill, exit code 137) and restarts it automatically
- The pod restarts, hits 24Mi again, crashes again — repeats indefinitely (CrashLoopBackOff)
- ArgoCD is not involved in this restart cycle at all

**The crash is still fully visible in logs — that's what matters:**
- Every OOMKill generates a Kubernetes event
- OTel DaemonSet ships that event to `logs-*` in Elasticsearch
- The agent finds it and correlates it with the `github-deployments` record
- The demo works even while the pod is in CrashLoopBackOff

**Timeline of what you see:**
```
push bad commit
    │
    ▼  (~5 seconds)
GitHub Actions: indexes commit to github-deployments
    │
    ▼  (~5 seconds)
ArgoCD: "OutOfSync" → syncing → "Synced" (green) ← ArgoCD did its job correctly
    │
    ▼  (meanwhile in the cluster)
paymentservice: Running → OOMKilled → CrashLoopBackOff
                          (Kubernetes restarts it — ArgoCD not involved)
    │
    ▼
OTel ships OOMKill event to logs-* in Elasticsearch
    │
    ▼
Agent: "itsBaivab deployed 24Mi limit at 14:28, pod crashed at 14:32 — that's the cause"
```

---

## Prerequisites — Install these first

Before starting, install and authenticate these tools on your local machine.

### gcloud CLI (Google Cloud)

```bash
# macOS
brew install --cask google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Authenticate
gcloud auth login
gcloud config set project gen-lang-client-0125290854
```

Verify:
```bash
gcloud projects list
# You should see: gen-lang-client-0125290854
```

### kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

### helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### argocd CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```

### gh CLI (GitHub)

```bash
# macOS
brew install gh

# Linux
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install gh -y

# Authenticate (opens browser)
gh auth login
# Select: GitHub.com → HTTPS → Login with a web browser → follow prompts
# Login as: itsBaivab
```

### git

```bash
# macOS — already installed, or:
brew install git

# Linux
sudo apt install git -y

# Configure your identity
git config --global user.name "itsBaivab"
git config --global user.email "your-email@example.com"
```

---

## 1. GKE Cluster

A GKE (Google Kubernetes Engine) cluster is the cloud environment where all 12 services run. Think of it as renting 3 powerful servers from Google that Kubernetes manages automatically.

### 1a. Create the cluster

```bash
gcloud container clusters create elastic-cloud-test \
  --zone us-central1-a \
  --project gen-lang-client-0125290854 \
  --no-enable-autoupgrade
```

> This takes 3-5 minutes. You'll see a progress spinner.

### 1b. Add the node pool

The default nodes are too small for everything we're running. Add larger nodes:

```bash
gcloud container node-pools create standard-pool \
  --cluster elastic-cloud-test \
  --zone us-central1-a \
  --project gen-lang-client-0125290854 \
  --machine-type e2-standard-4 \
  --num-nodes 3
```

> **Why e2-standard-4?** Online Boutique (12 services) + OTel collectors + ArgoCD need ~6 vCPU total.
> e2-standard-2 nodes only have ~940m allocatable CPU each — pods go Pending.
> e2-standard-4 gives ~1900m per node × 3 nodes = ~5700m total — plenty of headroom.

### 1c. Connect kubectl to the cluster

```bash
gcloud container clusters get-credentials elastic-cloud-test \
  --zone us-central1-a \
  --project gen-lang-client-0125290854
```

### 1d. Verify

```bash
kubectl get nodes
# Expect output like:
# NAME                                          STATUS   ROLES    AGE   VERSION
# gke-elastic-cloud-test-standard-pool-xxx-...  Ready    <none>   1m    v1.32.x
# gke-elastic-cloud-test-standard-pool-xxx-...  Ready    <none>   1m    v1.32.x
# gke-elastic-cloud-test-standard-pool-xxx-...  Ready    <none>   1m    v1.32.x
# All 3 nodes must say Ready
```

**Cluster details:**

| Field | Value |
|-------|-------|
| Cluster name | `elastic-cloud-test` |
| Zone | `us-central1-a` |
| Project | `gen-lang-client-0125290854` |
| Node pool | `standard-pool` |
| Machine type | `e2-standard-4` |
| Node count | 3 |

---

## 2. Online Boutique

Google's open-source microservices demo — a fake e-commerce store with 12 services (frontend, cart, payment, etc.). Each service runs as its own pod. A built-in load generator constantly sends traffic, which is what triggers OOMKills when we shrink memory.

**Repo:** `https://github.com/itsBaivab/microservices-demo`

The repo has been stripped down to just two things:
```
release/
  kubernetes-manifests.yaml   ← ArgoCD watches this file — edit this to break/fix things
.github/workflows/
  index-deploy.yml            ← runs on every push — indexes commit to Elasticsearch
```

### ⚠️ Important — commit changes BEFORE creating ArgoCD

The manifest in this repo already has all the required modifications:
- `strategy: type: Recreate` on the four breakable services
- Reduced CPU requests so pods fit on e2-standard-4 nodes
- `istio-manifests.yaml` removed (we don't run Istio)

**ArgoCD's first sync deploys exactly what is in the repo at that moment.** If you fork this repo and make changes locally without pushing first, ArgoCD will deploy the wrong version and you'll immediately have an OutOfSync conflict that's painful to resolve.

**Always follow this order:**
```
1. Make any changes to release/kubernetes-manifests.yaml
2. git add + git commit + git push  ← repo must reflect your desired state
3. THEN create the ArgoCD Application
4. ArgoCD first sync → deploys your version correctly
```

If you skip step 2, ArgoCD deploys the old version → cluster doesn't match repo → OutOfSync from day one.

### 2a. Clone the repo

```bash
git clone https://github.com/itsBaivab/microservices-demo
cd microservices-demo
```

### 2b. Breakable services

These four services have `strategy: type: Recreate` — when you push a change, the old pod dies immediately before the new one starts (visible outage):

| Service | Normal Memory Limit | Break value |
|---------|--------------------|-|
| paymentservice | 128Mi | 24Mi |
| cartservice | 128Mi | 32Mi |
| frontend | 64Mi | 32Mi |
| recommendationservice | 220Mi | 48Mi |

### 2c. How to create the problem (break paymentservice)

Open `release/kubernetes-manifests.yaml` in a text editor. Search for `name: paymentservice` under `kind: Deployment` (around line 631). Find the `resources` section:

```yaml
        resources:
          requests:
            cpu: 100m
            memory: 64Mi    ← change this to 24Mi
          limits:
            cpu: 200m
            memory: 128Mi   ← change this to 24Mi
```

Change **both** `memory` values to `24Mi` (requests must be ≤ limits — Kubernetes rejects it otherwise):

```yaml
        resources:
          requests:
            cpu: 100m
            memory: 24Mi
          limits:
            cpu: 200m
            memory: 24Mi
```

Then push:
```bash
git add release/kubernetes-manifests.yaml
git commit -m "perf: tune paymentservice memory limits for cost optimisation"
git push origin main
```

What happens automatically within ~10 seconds:
1. **GitHub Actions** → indexes the commit to `github-deployments` in Elasticsearch
2. **GitHub Webhook** → notifies ArgoCD of the new commit instantly
3. **ArgoCD** → pulls the change, applies the updated manifest to the cluster
4. **Recreate strategy** → Kubernetes kills the old paymentservice pod, starts new one with 24Mi
5. **OOMKill** → new pod hits the 24Mi limit under loadgenerator traffic and crashes
6. **OTel DaemonSet** → ships the crash event to `logs-*` in Elastic Cloud

Watch live:
```bash
kubectl get pods -n online-boutique -w
# You'll see paymentservice go: ContainerCreating → Running → OOMKilled → CrashLoopBackOff
# Press Ctrl+C to stop watching
```

### 2d. How to fix it

```bash
git revert HEAD --no-edit
git push origin main
# ArgoCD picks it up → restores 128Mi → pod recovers in ~30 seconds
```

---

## 3. ArgoCD

ArgoCD is a GitOps controller. It watches your GitHub repo and automatically applies any changes to the Kubernetes cluster. Every push to `main` triggers a deployment — no manual `kubectl apply` needed.

**ArgoCD UI:** `https://136.116.115.80`
**Username:** `admin`
**Password:** `GoDWfO9NiMWzTD8W`

### 3a. Install ArgoCD on the cluster

```bash
# Create a dedicated namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait until the server is ready (takes ~2 minutes)
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s

# Expose ArgoCD via a public IP (LoadBalancer)
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'

# Wait ~1 minute then get the public IP
kubectl get svc argocd-server -n argocd
# Look for the EXTERNAL-IP column — that's your ArgoCD URL
```

Get the admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
# Copy this password — you'll need it to log in
```

### 3b. Log in to ArgoCD CLI

```bash
argocd login <EXTERNAL-IP> --username admin --password <PASSWORD-FROM-ABOVE> --insecure
# Example: argocd login 136.116.115.80 --username admin --password GoDWfO9NiMWzTD8W --insecure
```

### 3c. Create the Application

This tells ArgoCD what repo to watch and where to deploy it:

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

> `automated: prune: true` — auto-deploys every push, removes resources deleted from the manifest.
> `selfHeal` is not set (defaults to false) — ArgoCD will NOT revert manual kubectl changes.

### 3d. Do the first sync

```bash
argocd app sync online-boutique --force --prune
# Wait for it to finish — you'll see all resources listed as Synced
```

Verify all 12 pods are running:
```bash
kubectl get pods -n online-boutique
# All 12 should show Running
# adservice, cartservice, checkoutservice, currencyservice, emailservice,
# frontend, loadgenerator, paymentservice, productcatalogservice,
# recommendationservice, redis-cart, shippingservice
```

Get the Online Boutique public URL:
```bash
kubectl get svc frontend-external -n online-boutique
# Open the EXTERNAL-IP in your browser — you should see the shop
```

### 3e. Set up GitHub Webhook (instant deploys)

By default ArgoCD polls GitHub every 3 minutes. The webhook makes it react in seconds.

```bash
# Generate a random secret
WEBHOOK_SECRET=$(openssl rand -hex 32)
echo "Your webhook secret: $WEBHOOK_SECRET"
# Save this somewhere — you'll need it if you recreate the webhook

# Store the secret in ArgoCD
kubectl patch secret argocd-secret -n argocd --type merge -p \
  "{\"stringData\": {\"webhook.github.secret\": \"$WEBHOOK_SECRET\"}}"

# Register the webhook on GitHub
gh api repos/itsBaivab/microservices-demo/hooks \
  --method POST \
  --field name=web \
  --field active=true \
  --field "events[]=push" \
  --field "config[url]=https://34.133.31.193/api/webhook" \
  --field "config[content_type]=json" \
  --field "config[secret]=$WEBHOOK_SECRET" \
  --field "config[insecure_ssl]=1"
# Expect output with "active": true and a webhook ID
```

Verify it works:
```bash
# Push an empty commit to trigger the webhook
cd microservices-demo
git commit --allow-empty -m "test: verify ArgoCD webhook"
git push origin main

# Watch ArgoCD logs for the webhook event
kubectl logs -n argocd deployment/argocd-server --since=30s | grep -i webhook
# Expect: "Received push event repo: https://github.com/itsBaivab/microservices-demo"
# Expect: "refreshing app 'online-boutique' from webhook"
```

### 3f. Navigate ArgoCD UI

**URL:** `https://136.116.115.80`
**Login:** `admin` / `GoDWfO9NiMWzTD8W`

- **Applications list** → you see `online-boutique` with a health status
- Click `online-boutique` → see all 12 services as boxes, color-coded by health
- **Sync Status: Synced** (green) = repo matches cluster
- **Sync Status: OutOfSync** (orange) = repo has changes not yet applied
- Click any service box → see the pod events, restart count, current image
- **APP DIFF** button → see exactly what changed between repo and cluster
- **SYNC** button → manually trigger a sync if auto-sync is slow

---

## 4. Elastic Cloud

Elastic Cloud is the managed Elasticsearch + Kibana service. All pod logs and metrics from the cluster ship here via OTel. This is also where the Agent Builder lives.

### 4a. Create an Elastic Cloud deployment

1. Go to `cloud.elastic.co` → sign in or create account
2. Click **Create deployment**
3. Choose:
   - **Cloud provider:** Google Cloud
   - **Region:** `us-central1` (same region as GKE — lower latency)
   - **Name:** `blame-the-deploy`
4. Click **Create deployment**
5. **IMPORTANT:** A dialog shows the `elastic` user password — **copy and save it now**, it is never shown again

Wait ~5 minutes for the deployment to be ready.

### 4b. Find your Elasticsearch endpoint

1. `cloud.elastic.co` → click your `blame-the-deploy` deployment
2. Under **Elasticsearch** → click **Copy endpoint**
3. It looks like: `https://abc123.us-central1.gcp.cloud.es.io:443`
4. Save this — you'll use it in every curl command and secret

**Current project endpoint:**
```
https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443
```

### 4c. Open Kibana

1. `cloud.elastic.co` → click your deployment → click **Open Kibana**
2. Or go directly to: `https://e177716345f94357bde1679d417f21ac.kb.us-central1.gcp.cloud.es.io`
3. Log in with username `elastic` and the password you saved in step 4a

### 4d. Create an API key

The API key is used by OTel, GitHub Actions, and curl commands to authenticate with Elasticsearch.

**Navigation:** Kibana → click the hamburger menu (☰) top left → **Stack Management** → under **Security** → **API Keys**

1. Click **Create API key**
2. **Name:** `blame-the-deploy`
3. Leave **Privileges** blank (unrestricted — fine for a demo)
4. Click **Create**
5. **IMPORTANT:** Copy the API key immediately — it is shown only once

The key looks like: `enVZTms1MEJnSzQyOEV6VzJQZHA6MG1TbTNyeHZEak84VGJvNVhwQXFPZw==`

Verify the key works:
```bash
curl -s -w "\nHTTP:%{http_code}" \
  -X GET "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443" \
  -H "Authorization: ApiKey <PASTE-YOUR-KEY-HERE>"
# Expect: HTTP:200 and a JSON response with cluster name
```

---

## 5. OpenTelemetry Kube-Stack

This ships all pod logs, metrics, and traces from the GKE cluster to Elastic Cloud automatically. It runs as:
- **DaemonSet collector** — one pod on every node, tails all container logs and collects node metrics
- **Cluster-stats collector** — collects cluster-wide Kubernetes metrics
- **Gateway collector** — batches everything and sends to Elastic Cloud over OTLP

Once installed, you never touch it again — it automatically picks up logs from any new pod in any namespace.

### 5a. Create the Kubernetes secret with ES credentials

```bash
kubectl create namespace opentelemetry-operator-system

kubectl create secret generic elastic-secret-otel \
  --namespace opentelemetry-operator-system \
  --from-literal=elastic_endpoint='https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443' \
  --from-literal=elastic_api_key='<PASTE-YOUR-API-KEY-HERE>'
```

### 5b. Install via Helm

```bash
# Add the OTel Helm chart repository
helm repo add open-telemetry 'https://open-telemetry.github.io/opentelemetry-helm-charts' --force-update

# Download the values file locally (direct URL times out)
curl -fsSL 'https://raw.githubusercontent.com/elastic/elastic-agent/refs/tags/v9.3.3/deploy/helm/edot-collector/kube-stack/values.yaml' \
  -o otel-values.yaml

# Install
helm upgrade --install opentelemetry-kube-stack open-telemetry/opentelemetry-kube-stack \
  --namespace opentelemetry-operator-system \
  --values otel-values.yaml \
  --version '0.12.4'
```

> **If Helm fails with "conflict" error:** the previous install left a broken state.
> Run: `helm uninstall opentelemetry-kube-stack -n opentelemetry-operator-system`
> Then re-run the install command above.

### 5c. Verify collectors are running

```bash
kubectl get pods -n opentelemetry-operator-system
# Expect all Running — should look like this:
# opentelemetry-kube-stack-cluster-stats-collector-xxx    1/1   Running
# opentelemetry-kube-stack-daemon-collector-xxx           1/1   Running   (×3, one per node)
# opentelemetry-kube-stack-gateway-collector-xxx          1/1   Running   (×2)
# opentelemetry-kube-stack-opentelemetry-operator-xxx     2/2   Running
```

Check logs are being shipped:
```bash
kubectl logs -n opentelemetry-operator-system \
  $(kubectl get pod -n opentelemetry-operator-system \
    -l app.kubernetes.io/component=opentelemetry-collector \
    -o name | grep gateway | head -1) | tail -5
# Look for lines like: "log records": 88 — this means logs are flowing to Elastic
```

### 5d. Verify data in Kibana

**Navigation:** Kibana → hamburger menu (☰) → **Discover**

1. Click the index pattern dropdown (top left, shows the current index name) → select **`logs-*`**
2. Set time range to **Last 15 minutes** (top right, looks like a calendar icon)
3. Click **Try ES|QL** (top right of the query bar) and run:

```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| SORT @timestamp DESC
| LIMIT 10
| KEEP @timestamp, resource.attributes.k8s.deployment.name, resource.attributes.k8s.pod.name, body.text
```

You should see rows of logs from the online-boutique pods.

> **Important — where pod metadata lives in OTel logs:**
> All Kubernetes metadata is under `resource.attributes`, not `attributes`:
> - `resource.attributes.k8s.namespace.name` — the namespace
> - `resource.attributes.k8s.deployment.name` — the service name (e.g. `paymentservice`)
> - `resource.attributes.k8s.pod.name` — the exact pod name
> - `resource.attributes.k8s.node.name` — which node the pod ran on
> - `body.text` — the actual log line text

**Check infrastructure metrics:**
- Kibana → hamburger menu → **Observability** → **Infrastructure** → **Kubernetes**
- You'll see CPU, memory, and pod counts per node and per deployment

---

## 6. GitHub Deployments Index

### Why this index exists

When a pod OOMKills, Elasticsearch has the crash log — pod name, time, error. But it has no idea **who changed what** to cause it. That information lives in GitHub.

`github-deployments` is a custom Elasticsearch index that captures every push to `main`:

```json
{
  "timestamp": "2026-04-16T08:58:00Z",
  "commit_sha": "a60608f2",
  "author": "itsBaivab",
  "service": "paymentservice",
  "change": "perf: tune paymentservice memory limits for cost optimisation",
  "diff_url": "https://github.com/itsBaivab/microservices-demo/compare/..."
}
```

The Agent Builder joins this with `logs-*`:
- Crash log: `paymentservice OOMKilled at 08:59`
- Deploy record: `paymentservice deployed at 08:58 by itsBaivab, changed memory limits`
- Agent conclusion: `itsBaivab's commit caused the crash — revert it`

### 6a. Create the index

Run this once to create the index with the correct field types:

```bash
curl -X PUT "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443/github-deployments" \
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
# Expect: {"acknowledged":true,"shards_acknowledged":true,"index":"github-deployments"}
```

Verify it exists in Kibana:
- Kibana → hamburger menu → **Stack Management** → **Index Management**
- `github-deployments` should appear in the list

### 6b. GitHub Actions workflow — how it works

The file `.github/workflows/index-deploy.yml` is already in the repo. It fires automatically on every push to `main` and:

1. Checks out the repo with the last 2 commits (so it can diff them)
2. Looks at what changed in `release/kubernetes-manifests.yaml`
3. Searches the changed lines for a known service name (`paymentservice`, `cartservice`, etc.)
4. POSTs a document to `github-deployments` with the commit SHA, author, message, and diff URL

The workflow needs two secrets to authenticate with Elasticsearch.

### 6c. Add GitHub secrets

**Via CLI:**
```bash
gh secret set ES_ENDPOINT \
  --repo itsBaivab/microservices-demo \
  --body "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443"

gh secret set ES_API_KEY \
  --repo itsBaivab/microservices-demo \
  --body "<YOUR-API-KEY>"
```

**Via GitHub UI:**
1. Go to `github.com/itsBaivab/microservices-demo`
2. Click **Settings** (top tab bar)
3. Left sidebar → **Secrets and variables** → **Actions**
4. Click **New repository secret**
5. Add `ES_ENDPOINT` = `https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443`
6. Add `ES_API_KEY` = your API key

Verify secrets are set:
```bash
gh secret list --repo itsBaivab/microservices-demo
# Expect: ES_API_KEY and ES_ENDPOINT listed with recent update dates
```

### 6d. Test the workflow

```bash
cd microservices-demo
git commit --allow-empty -m "test: verify ES indexing"
git push origin main
```

Check it ran:
```bash
gh run list --repo itsBaivab/microservices-demo --limit 3
# Expect: "Index deployment to Elasticsearch" → Status: completed → Conclusion: success
```

Verify data landed:
```bash
curl -s "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443/github-deployments/_search?sort=timestamp:desc&size=3" \
  -H "Authorization: ApiKey <YOUR-API-KEY>" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for h in d['hits']['hits']:
  s = h['_source']
  print(f\"commit: {s.get('commit_sha','?')[:8]} | author: {s.get('author','?')} | service: {s.get('service','?')} | change: {s.get('change','?')[:60]}\")
"
```

Or check in Kibana:
- Kibana → hamburger menu → **Discover** → index pattern → select **`github-deployments`**
- You should see the commit records

---

## 7. Agent Builder — Blame Tool

The Agent Builder creates an AI agent that can query Elasticsearch and reason over the results. Our agent has two tools — one to find crash logs, one to find deploy history — and it correlates them to answer "who broke the pod?"

### 7a. Set up an LLM connector

The agent needs an LLM to reason over query results. You need an OpenAI API key.

**Navigation:** Kibana → hamburger menu (☰) → **Stack Management** → **Connectors**

1. Click **Create connector**
2. Scroll to **Generative AI** section → select **OpenAI**
3. Fill in:
   - **Connector name:** `openai-blame`
   - **OpenAI API key:** your OpenAI API key (from `platform.openai.com` → API Keys → Create new)
   - **Default model:** `gpt-4o`
4. Click **Save & test** → you should see a test response

> **Don't have OpenAI?** You can also use:
> - **Amazon Bedrock** (select Bedrock instead of OpenAI, use AWS credentials)
> - **Google Gemini** (select Google Gemini, use a Google AI API key)

### 7b. Open Agent Builder

**Navigation:** Kibana → hamburger menu (☰) → **Search** → **Search AI Lake** → **Agent Builder**

> If you don't see **Search AI Lake**: your Kibana version may not have it.
> Alternative: hamburger menu → **Observability** → **AI Assistant** — simpler interface, same capability.

### 7c. Create the agent

1. Click **Create agent** (or **New agent**)
2. Fill in:
   - **Name:** `blame-the-deploy`
   - **LLM connector:** select `openai-blame` (the connector you just created)
3. Paste this as the **Instructions** (system prompt):

```
You are an SRE assistant. When asked why a service is crashing:
1. Use get_crash_logs to find OOMKill or memory events in Elasticsearch logs
2. Use get_deploy_history to find the most recent GitHub deployment to that service
3. If the deploy happened shortly before the crash, that is the likely cause
4. Report: crash time, commit SHA, author, what changed, and exactly how to fix it
Always cite the commit SHA and author name in your answer.
```

### 7d. Add Tool 1 — Crash Logs

Click **Add tool** → **Elasticsearch query**:

| Field | Value |
|-------|-------|
| Tool name | `get_crash_logs` |
| Description | `Search pod logs for OOMKill or memory crash events` |
| Index | `logs-*` |
| Query type | ES\|QL |

Query:
```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| WHERE body.text LIKE "*OOMKill*" OR body.text LIKE "*memory*"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, resource.attributes.k8s.deployment.name, resource.attributes.k8s.pod.name, body.text
```

### 7e. Add Tool 2 — Deploy History

Click **Add tool** → **Elasticsearch query**:

| Field | Value |
|-------|-------|
| Tool name | `get_deploy_history` |
| Description | `Get recent GitHub deployments to find who deployed what and when` |
| Index | `github-deployments` |
| Query type | ES\|QL |

Query:
```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 5
| KEEP timestamp, author, commit_sha, service, change, diff_url
```

### 7f. Save and test

Click **Save**. Then in the agent chat panel, type:

```
Why is paymentservice crashing? Check the logs and recent deployments to find who caused it.
```

Expected response:
```
OOMKill detected on paymentservice at 08:59 UTC.

Most recent deployment to paymentservice:
  Commit: a60608f2 by @itsBaivab at 08:58 UTC
  Change: "perf: tune paymentservice memory limits for cost optimisation"
  Diff: https://github.com/itsBaivab/microservices-demo/compare/...

Root cause: Memory limit was reduced to 24Mi. The pod hits the limit
immediately under traffic from the loadgenerator and gets OOMKilled.

Fix: git revert a60608f2 && git push origin main
ArgoCD will pick up the fix and redeploy within seconds.
```

---

## 8. Full Demo Loop

Complete end-to-end sequence for running the demo. Takes about 3 minutes from push to agent answer.

### Step 1 — Confirm everything is healthy

```bash
kubectl get pods -n online-boutique
# All 12 pods should show Running with 0 restarts
```

Open `http://34.59.76.90` in browser — the shop should load.

In Kibana → Discover → `logs-*` → confirm you see recent logs from `online-boutique`.

### Step 2 — Push the bad commit

Open `release/kubernetes-manifests.yaml`. Find the paymentservice Deployment (search for `name: paymentservice` under `kind: Deployment`, around line 631). Change **both** memory values to `24Mi`:

```yaml
        resources:
          requests:
            cpu: 100m
            memory: 24Mi    ← was 64Mi
          limits:
            cpu: 200m
            memory: 24Mi    ← was 128Mi
```

Then:
```bash
git add release/kubernetes-manifests.yaml
git commit -m "perf: tune paymentservice memory limits for cost optimisation"
git push origin main
```

Watch the pod die:
```bash
kubectl get pods -n online-boutique -w
# paymentservice: ContainerCreating → Running → OOMKilled → CrashLoopBackOff
```

Check ArgoCD UI → `online-boutique` → the app shows **Synced** (ArgoCD applied the change correctly) but paymentservice is **Degraded** (the pod is crashing).

### Step 3 — Ask the agent

Kibana → hamburger menu → **Search** → **Search AI Lake** → **Agent Builder** → open `blame-the-deploy`

Type:
```
Why is paymentservice crashing? Check the logs and recent deployments to find who caused it.
```

### Step 4 — Fix it

```bash
git revert HEAD --no-edit
git push origin main
```

ArgoCD picks up the fix within seconds → pod restores → ask the agent again:
```
Is paymentservice healthy now?
```

---

## 9. Quick Reference

### All endpoints and credentials

| Service | URL | Login |
|---------|-----|-------|
| Online Boutique | `http://34.59.76.90` | none |
| ArgoCD UI | `https://136.116.115.80` | `admin` / `GoDWfO9NiMWzTD8W` |
| Kibana | `https://e177716345f94357bde1679d417f21ac.kb.us-central1.gcp.cloud.es.io` | `elastic` / saved from deployment creation |
| Elasticsearch API | `https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443` | API key |
| GitHub repo | `https://github.com/itsBaivab/microservices-demo` | GitHub account |
| Elastic Cloud console | `https://cloud.elastic.co` | Google/email account |

### Essential commands

```bash
# Check pod status
kubectl get pods -n online-boutique
kubectl get pods -n argocd
kubectl get pods -n opentelemetry-operator-system

# Watch pods live
kubectl get pods -n online-boutique -w

# Check why a pod crashed (look for OOMKilled, Exit Code 137)
kubectl describe pod -n online-boutique <pod-name> | grep -A5 'Last State\|OOM\|Reason\|Exit Code'

# ArgoCD status
argocd app get online-boutique

# Force ArgoCD sync manually
argocd app sync online-boutique --force --prune

# Check OTel is shipping logs (look for "log records": N)
kubectl logs -n opentelemetry-operator-system \
  $(kubectl get pod -n opentelemetry-operator-system \
    -l app.kubernetes.io/component=opentelemetry-collector \
    -o name | grep gateway | head -1) | tail -10

# Check how many docs are in github-deployments
curl -s "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443/github-deployments/_count" \
  -H "Authorization: ApiKey <YOUR-API-KEY>"

# See the last 5 deployment records
curl -s "https://e177716345f94357bde1679d417f21ac.us-central1.gcp.cloud.es.io:443/github-deployments/_search?sort=timestamp:desc&size=5" \
  -H "Authorization: ApiKey <YOUR-API-KEY>" \
  | python3 -c "
import sys,json
for h in json.load(sys.stdin)['hits']['hits']:
  s=h['_source']
  print(s.get('commit_sha','')[:8], '|', s.get('author',''), '|', s.get('service',''), '|', s.get('change','')[:60])
"

# Check GitHub Actions workflow ran
gh run list --repo itsBaivab/microservices-demo --limit 5
```

### Kibana navigation cheat sheet

| What you want | Exact path |
|---------------|------------|
| Pod logs | ☰ → Discover → index: `logs-*` → filter by `resource.attributes.k8s.namespace.name: online-boutique` |
| OOMKill events | ☰ → Discover → Try ES\|QL → run crash logs query below |
| Deploy history | ☰ → Discover → index: `github-deployments` |
| Infrastructure metrics | ☰ → Observability → Infrastructure → Kubernetes |
| Agent Builder | ☰ → Search → Search AI Lake → Agent Builder |
| LLM connectors | ☰ → Stack Management → Connectors |
| API Keys | ☰ → Stack Management → Security → API Keys |
| Index list | ☰ → Stack Management → Index Management |
| GitHub Actions runs | `github.com/itsBaivab/microservices-demo` → Actions tab |
| ArgoCD app view | `https://136.116.115.80` → login → click `online-boutique` |

### ES|QL queries for manual investigation

**Find OOMKill events:**
```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| WHERE body.text LIKE "*OOMKill*" OR body.text LIKE "*memory*"
| SORT @timestamp DESC
| LIMIT 20
| KEEP @timestamp, resource.attributes.k8s.deployment.name, resource.attributes.k8s.pod.name, body.text
```

**Find recent GitHub deploys:**
```esql
FROM github-deployments
| SORT timestamp DESC
| LIMIT 10
| KEEP timestamp, author, commit_sha, service, change
```

**Find all logs from a specific service:**
```esql
FROM logs-*
| WHERE resource.attributes.k8s.deployment.name == "paymentservice"
| SORT @timestamp DESC
| LIMIT 50
| KEEP @timestamp, body.text
```

**Find logs from a specific time window:**
```esql
FROM logs-*
| WHERE resource.attributes.k8s.namespace.name == "online-boutique"
| WHERE @timestamp >= NOW() - 1 hour
| SORT @timestamp DESC
| LIMIT 100
| KEEP @timestamp, resource.attributes.k8s.deployment.name, body.text
```
