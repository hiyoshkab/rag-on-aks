# RAG on AKS
This is a retrieval-augmented question-answering application (RAG) for plain-text documents built on AKS. Upload a UTF-8 `.txt` or `.md` file, store its embeddings in Qdrant, and ask questions answered with Azure AI Foundry. The React frontend displays the generated answer and the source passages used to produce it.

The application runs in an AKS virtual network, and the Azure AI Foundry
account is configured with private network access. The AKS API server and
application exposure should still be reviewed separately for each environment.

## Architecture
![RAG architecture](./architecture.png)

The backend chunks uploaded text, creates 384-dimensional embeddings with `BAAI/bge-small-en-v1.5`, and stores them in the `documents` Qdrant collection. Queries retrieve the most relevant chunks and pass them to the configured Azure
OpenAI deployment to generate an answer.

## Demo
![Demo](./demo.png)

## Repository layout
| Path | Description |
| --- | --- |
| [`backend/`](./backend) | FastAPI API, ingestion and retrieval logic, and Python tests |
| [`frontend/app_frontend/`](./frontend/app_frontend) | React and TypeScript single-page application |
| [`frontend/Dockerfile`](./frontend/Dockerfile) | Multi-stage frontend image served by Nginx |
| [`backend/Dockerfile`](./backend/Dockerfile) | FastAPI container image |
| [`helm/raas/`](./helm/raas) | Helm chart for the frontend, API, and Qdrant |
| [`terraform/`](./terraform) | Azure infrastructure and workload identity configuration |
| [`.github/workflows/`](./.github/workflows) | Test, image build, Terraform, and deployment workflows |

## Prerequisites

- Azure CLI, authenticated to the subscription containing the AKS cluster
- `kubectl`
- `kubelogin` for Microsoft Entra authentication to AKS
- An AKS cluster and RaaS deployment created using the [Terraform and Helm
  deployment instructions](#azure-and-kubernetes-deployment)

## Azure and Kubernetes deployment
Deployment is done through Github Actions.
Infrastructure is managed with Terraform and application resources with Helm. Read the detailed guides before deploying:

- [Terraform setup](./terraform/README.md), including remote state and Foundry provisioning
- [Helm deployment](./helm/raas/README.md), including required GitHub variables and secrets

The deployment uses an AKS cluster, a Qdrant StatefulSet with persistent storage, and Azure Workload Identity for the backend's user-assigned identity. The application namespace is `raas`.

On pushes to `main` that change `backend/**` or `frontend/**`, the deployment
workflow runs the corresponding tests, builds commit-tagged images, pushes them
to Docker Hub, and deploys the Helm release. Manual workflow dispatch supports
forcing frontend/backend builds or deployment. Terraform changes use the
separate plan/apply workflow.

> Note: For production, Azure Container Registry with Private Endpoint should be used instead of Docker Hub. Docker Hub is used here for simpler design in this demo. 

## Run the deployed application

The Helm chart deploys the frontend and backend as internal Kubernetes
`ClusterIP` services. Connect to AKS with your Microsoft Entra identity, then
forward the frontend service to your local machine.

### 1. Connect `kubectl` to AKS
```sh
CLUSTER_NAME="<AKS cluster name>"
RESOURCE_GROUP="<AKS resource group>"

az login
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
```

If you are using a different shell or machine, replace the variables with the
values from `terraform output`. The cluster disables local administrator
accounts, so use the Entra-enabled kubeconfig rather than `--admin`.

### 2. Check the deployment

```sh
kubectl get pods,svc -n raas
kubectl get events -n raas --sort-by=.lastTimestamp
```

Wait until the frontend and backend pods report `Running` and `READY 1/1`.
Check application health and backend logs when troubleshooting:

```sh
kubectl logs -n raas deploy/app-backend
kubectl logs -n raas deploy/app-frontend
kubectl run curl --rm -it --restart=Never -n raas \
  --image=curlimages/curl -- \
  curl -fsS http://app-backend-service:8000/healthz
```

### 3. Open the application

Forward the internal frontend service to port 8080:

```sh
kubectl port-forward -n raas svc/app-frontend-service 8080:80
```

Open <http://127.0.0.1:8080> in a browser. The frontend proxies `/ingest` and
`/query` to the backend service inside the cluster, so no separate backend
port-forward is needed.

You can also call the application API through the forwarded frontend:

```sh
curl -fsS http://127.0.0.1:8080/healthz
curl -F "file=@sample.txt" http://127.0.0.1:8080/ingest
curl -X POST http://127.0.0.1:8080/query \
  -H 'Content-Type: application/json' \
  -d '{"question":"What does the document say?","top_k":3}'
```

Keep the `kubectl port-forward` command running while using the application.
Press `Ctrl+C` to stop forwarding.

> Note: `kubectl port-forward` is intended only for local development and troubleshooting. In production, expose the frontend through an HTTPS ingress, such as Azure Application Gateway for Containers or an ingress controller, and map it to a DNS hostname.

### 4. Open the Qdrant dashboard

Qdrant is deployed as an internal `ClusterIP` service. In another terminal,
forward its HTTP port:

```sh
kubectl port-forward -n raas svc/qdrant 6333:6333
```

Open <http://127.0.0.1:6333/dashboard> to inspect collections and points. You
can also check the API:

```sh
curl http://127.0.0.1:6333/collections
```

Keep the port-forward running while using the dashboard. Qdrant is not
publicly exposed by the Helm chart.

## Configuration
The backend reads these environment variables:

| Variable | Default | Description |
| --- | --- | --- |
| `AZURE_OPENAI_ENDPOINT` | empty | Azure AI Foundry OpenAI-compatible endpoint |
| `AZURE_OPENAI_DEPLOYMENT` | `gpt-5.4-mini` | Chat model deployment name |
| `LLM_TIMEOUT` | `300` | LLM request timeout in seconds |
| `QDRANT_URL` | `http://localhost:6333` | Qdrant endpoint |
| `COLLECTION` | `documents` | Qdrant collection name |
| `EMBED_MODEL` | `BAAI/bge-small-en-v1.5` | FastEmbed model |
| `EMBED_DIM` | `384` | Expected embedding dimension |
| `EMBED_BATCH_SIZE` | `64` | Embedding batch size |
| `MAX_UPLOAD_BYTES` | `5242880` | Maximum upload size |
| `CHUNK_SIZE` | `256` | Chunk size used during ingestion |
| `OVERLAP` | `32` | Chunk overlap |

## API
### `POST /ingest`

Accepts a multipart form upload:
```sh
curl -F "file=@sample.txt" http://127.0.0.1:8080/ingest
```

Only UTF-8 text is accepted. The response includes the normalized filename and
number of stored chunks.

### `POST /query`
Accepts JSON with a question and optional number of results:

```sh
curl -X POST http://127.0.0.1:8080/query \
  -H 'Content-Type: application/json' \
  -d '{"question":"What does the document say?","top_k":3}'
```

The response contains the generated answer and the source filename, chunk
index, relevance score, and text for each retrieved passage.
