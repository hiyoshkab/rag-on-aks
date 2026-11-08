# RAAS Helm chart

This chart installs the RaaS frontend, backend, and Qdrant StatefulSet into the
`raas` namespace. The app pipeline runs the relevant frontend and backend tests,
builds and pushes commit-tagged images for changed components, then calls the
reusable Helm workflow to deploy them. The reusable workflow can also be started
by the application workflow; when image tags are omitted, it reuses the image
tags stored in the currently installed Helm release. The application workflow's
manual dispatch can force builds or deployment.

## Deploy from GitHub Actions

The workflow expects these GitHub repository variables:

- `DOCKERHUB_REPO`: fully qualified Docker image repository prefix used by the
  build workflow, for example `docker.io/YOUR_DOCKERHUB_USERNAME`.
- `AKS_CLUSTER_NAME`: the AKS cluster name.
- `AKS_RESOURCE_GROUP`: the Azure resource group containing the cluster.
- `AZURE_OPENAI_ENDPOINT`: endpoint provided to the app's ConfigMap at deploy
  time.
- `AZURE_APP_CLIENT_ID`: client ID from Terraform's `app_identity_client_id`
  output. It configures the app's Kubernetes service account for Azure
  Workload Identity.
It expects these GitHub Actions secrets:

- `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`: credentials used to push the
  image to Docker Hub. Use a Docker Hub access token, not the account password.
- `AZURE_CREDENTIALS`: the Azure service principal credentials JSON consumed by
  `azure/login`. The service principal must be able to retrieve credentials for
  the AKS cluster and deploy resources to the `raas` namespace.

The AKS Terraform configuration disables local accounts, so the Azure identity also needs the Azure Kubernetes Service Cluster User role and suitable Kubernetes/Azure RBAC permissions. The deploy workflow installs `kubelogin` and configures the AKS context to use it for Entra authentication. The app uses a separate user-assigned identity federated to its Kubernetes service account; it does not inherit the node identity's Azure permissions.