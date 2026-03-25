# OpenRAG on OpenShift — Deployment Guide

## Prerequisites

- OpenShift 4.x cluster with `oc` CLI
- Helm 3.x
- Namespace with sufficient quota (16Gi+ memory for Docling)

## Container Images

All OpenRAG images are built from UBI9 base images and pushed to a single
Quay.io repository with component-prefixed tags:

| Component | Image Tag | Base |
|-----------|-----------|------|
| Backend | `quay.io/opendatahub/odh-openrag:backend-latest` | UBI9 minimal |
| Langflow | `quay.io/opendatahub/odh-openrag:langflow-latest` | UBI9 minimal |
| OpenSearch | `quay.io/opendatahub/odh-openrag:opensearch-latest` | UBI9 |

The Helm chart init containers use `curlimages/curl` and `busybox` (not UBI9).
These are ephemeral and run only during pod initialization.

Docling Serve uses the upstream image `ghcr.io/docling-project/docling-serve:v1.5.0`.

Images are rebuilt automatically via GitHub Actions on push to `main`.

## Step 1: Create Secrets

```bash
NAMESPACE=my-namespace

# OpenSearch admin password
oc create secret generic openrag-opensearch \
  --from-literal=password="$(openssl rand -base64 24)" \
  -n $NAMESPACE

# OpenAI API key (required for embeddings and LLM)
# Get your key from https://platform.openai.com/api-keys
oc create secret generic openrag-llm-providers \
  --from-literal=openai-api-key="sk-your-key-here" \
  -n $NAMESPACE
```

## Step 2: Deploy OpenSearch and Docling

These are prerequisites — the Helm chart does not deploy them.

```bash
oc apply -f hack/opensearch.yaml -n $NAMESPACE
oc apply -f hack/docling.yaml -n $NAMESPACE

# Wait for both to be ready
oc wait --for=condition=ready pod -l app=openrag-opensearch --timeout=120s -n $NAMESPACE
oc wait --for=condition=ready pod -l app=openrag-docling --timeout=120s -n $NAMESPACE
```

## Step 3: Create Helm Values

Create a file `my-values.yaml`:

```yaml
global:
  imageRegistry: "quay.io/opendatahub/odh-openrag"
  imageTag: "latest"

  opensearch:
    host: "openrag-opensearch"
    port: 9200
    scheme: "https"
    username: "admin"
    password: ""  # Will be read from secret created in step 1
    indexName: "documents"

  docling:
    host: openrag-docling
    port: 5001
    scheme: "http"

langflow:
  image:
    repository: quay.io/opendatahub/odh-openrag
    tag: "langflow-latest"

  auth:
    autoLogin: false
    superuser: "admin"
    superuserPassword: "change-me-to-a-strong-password"
    secretKey: ""  # Generate with: python3 -c "import base64,os; print(base64.urlsafe_b64encode(os.urandom(32)).decode())"

  doclingServeUrl: "http://openrag-docling:5001"

  # emptyDir avoids RWO PVC multi-attach issues between langflow and backend
  persistence:
    enabled: false

  flows:
    loadDefaults: true
    git:
      enabled: true
      owner: cooktheryan
      repo: openrag
      branch: main

backend:
  image:
    repository: quay.io/opendatahub/odh-openrag
    tag: "backend-latest"

  auth:
    sessionSecret: ""  # Generate with: openssl rand -hex 32

  features:
    disableIngestWithLangflow: true  # Use direct ingestion (more reliable)
    ingestSampleData: true

frontend:
  enabled: false

dashboards:
  enabled: false

ingress:
  enabled: false

llmProviders:
  openai:
    enabled: true
    apiKey: ""  # Will be read from secret created in step 1

appConfig:
  knowledge:
    embeddingModel: "text-embedding-3-small"
    embeddingProvider: "openai"

# OpenShift: let SCC assign UID/GID
podSecurityContext:
  runAsNonRoot: true
  fsGroup: null

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: false
  runAsUser: null
  runAsGroup: null

serviceAccount:
  create: true
```

**Important:** Set real values for:
- `langflow.auth.superuserPassword`
- `langflow.auth.secretKey` (must be 32 url-safe base64-encoded bytes)
- `backend.auth.sessionSecret`
- `global.opensearch.password` (must match the secret from step 1)

The `llmProviders.openai.apiKey` is stored in the Helm-managed secret. Pass
it via `--set` to avoid committing it:

```bash
OPENAI_KEY="sk-your-key-here"
```

## Step 4: Install with Helm

```bash
helm install openrag kubernetes/helm/openrag \
  -n $NAMESPACE \
  -f my-values.yaml \
  --set llmProviders.openai.apiKey="$OPENAI_KEY" \
  --set global.opensearch.password="$(oc get secret openrag-opensearch -n $NAMESPACE -o jsonpath='{.data.password}' | base64 -d)"
```

## Step 5: Verify

```bash
# Wait for pods
oc get pods -n $NAMESPACE -w

# Create an API key
oc exec -n $NAMESPACE deploy/openrag-backend -c backend -- \
  curl -s -X POST http://localhost:8000/keys \
  -H "Content-Type: application/json" \
  -d '{"name": "my-key"}'

# Test search
API_KEY="orag_..."  # from above
oc exec -n $NAMESPACE deploy/openrag-backend -c backend -- \
  curl -s -X POST http://localhost:8000/v1/search \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "What is OpenRAG?", "limit": 3}'
```

## Upgrading

```bash
helm upgrade openrag kubernetes/helm/openrag \
  -n $NAMESPACE \
  -f my-values.yaml \
  --set llmProviders.openai.apiKey="$OPENAI_KEY" \
  --set global.opensearch.password="$(oc get secret openrag-opensearch -n $NAMESPACE -o jsonpath='{.data.password}' | base64 -d)"
```

## Rebuilding Images

Images are built automatically by GitHub Actions (`.github/workflows/build-quay.yml`)
on push to `main`. To build manually:

```bash
# Backend
docker build --platform linux/amd64 -t quay.io/opendatahub/odh-openrag:backend-latest -f Dockerfile.backend .

# Langflow
docker build --platform linux/amd64 -t quay.io/opendatahub/odh-openrag:langflow-latest -f Dockerfile.langflow .

# OpenSearch
docker build --platform linux/amd64 -t quay.io/opendatahub/odh-openrag:opensearch-latest -f Dockerfile .
```

## Uninstalling

```bash
helm uninstall openrag -n $NAMESPACE
oc delete -f hack/opensearch.yaml -n $NAMESPACE
oc delete -f hack/docling.yaml -n $NAMESPACE
oc delete pvc -l app.kubernetes.io/instance=openrag -n $NAMESPACE
oc delete pvc openrag-opensearch-data -n $NAMESPACE
```
