# OpenShift Deploy Runbook

## Prerequisites

- **OpenShift** 4.14+ with `oc` CLI configured
- **Helm** 3.x installed
- **NGC API Key** — for pulling container images from `nvcr.io`. Get one at https://org.ngc.nvidia.com/setup/api-keys
- **NVIDIA API Key** — for hosted LLM inference on `integrate.api.nvidia.com`. Get one at https://build.nvidia.com (click "Get API Key" on any model page)
- **Tavily API Key** (optional) — for web search functionality. Get a free key at https://tavily.com. Without it, web search queries will return empty results.

> **Important:** The NGC API Key and NVIDIA API Key are **different keys** from different portals. Using the wrong key for `ngc-secret` will cause `ImagePullBackOff` errors.

## 1. Login to OpenShift cluster

```bash
oc login --token=$OPENSHIFT_TOKEN --server=$OPENSHIFT_CLUSTER_URL
```

## 2. Create namespace

```bash
oc new-project ns-aiq
```

> **Namespace / appname mapping:** The Helm chart derives the target namespace from `aiq.appname` (default: `aiq`) by prefixing `ns-`, resulting in `ns-aiq`. If you use a custom namespace, set `--set aiq.appname=<your-namespace> --set aiq.project.deploymentTarget=kind` at install time so the chart targets your namespace directly.

## 3. Create secrets

Export your API keys before running these commands:

```bash
export NGC_API_KEY="<your NGC org key>"
export NVIDIA_API_KEY="<your build.nvidia.com key>"
export TAVILY_API_KEY="<your Tavily key>"   # optional
```

Create the secrets (safe to re-run — deletes existing secrets first):

```bash
oc delete secret ngc-secret -n ns-aiq --ignore-not-found
oc create secret docker-registry ngc-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="$NGC_API_KEY" \
  -n ns-aiq

oc delete secret aiq-credentials -n ns-aiq --ignore-not-found
oc create secret generic aiq-credentials \
  --from-literal=NVIDIA_API_KEY="$NVIDIA_API_KEY" \
  --from-literal=TAVILY_API_KEY="${TAVILY_API_KEY:-placeholder}" \
  --from-literal=DB_USER_NAME="aiq" \
  --from-literal=DB_USER_PASSWORD="aiq_dev" \
  -n ns-aiq
```

## 4. Deploy the application

```bash
helm dependency build deploy/helm/deployment-k8s

helm upgrade --install aiq deploy/helm/deployment-k8s \
  -f deploy/helm/deployment-k8s/values-openshift.yaml \
  -n ns-aiq \
  --wait --timeout 10m
```

> For a custom namespace, add: `--set aiq.appname=<your-namespace> --set aiq.project.deploymentTarget=kind`

## 5. Verify the deployment

```bash
# All pods should be Running 1/1
oc get pods -n ns-aiq

# Get the frontend URL
echo "https://$(oc get route aiq-frontend -n ns-aiq -o jsonpath='{.spec.host}')"

# Backend health check
oc exec deploy/aiq-backend -n ns-aiq -- wget -qO- http://localhost:8000/health

# Stream backend logs
oc logs -f deploy/aiq-backend -n ns-aiq -c backend
```

**Expected pods:**

| Pod | Purpose |
|-----|---------|
| `aiq-backend` | Research assistant API (hosted LLMs via integrate.api.nvidia.com) |
| `aiq-frontend` | Web UI |
| `aiq-postgres` | PostgreSQL for job state, checkpoints, and summaries |

## 6. Uninstall

```bash
helm uninstall aiq -n ns-aiq
oc delete pvc --all -n ns-aiq    # deletes PostgreSQL data
oc delete project ns-aiq
```