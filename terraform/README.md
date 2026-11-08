# Terraform

This configuration creates the Azure infrastructure required by RaaS:

- An AKS cluster with Azure RBAC, OIDC, and Workload Identity enabled
- A virtual network and subnets for AKS and Azure AI Foundry
- An Azure AI Foundry account and model deployment
- A private endpoint and private DNS integration for Foundry
- A user-assigned identity for the backend
- Azure role assignments for the backend and GitHub Actions deployment identity

## Prerequisites

- Terraform 1.16.5 or newer
- Azure CLI
- An Azure subscription with permission to create resource groups, AKS,
  networking, Azure AI Foundry, private endpoints, managed identities, and
  role assignments
- Permission to create the Foundry resource group, account, model deployment,
  private endpoint, and private DNS resources
- A Microsoft Entra group object ID for AKS administrators
- The Microsoft Entra service principal object ID used by GitHub Actions

Terraform creates the resource group named by `foundry_resource_group_name` for
the Foundry account and private networking. A separate generated resource
group contains the AKS and application resources.

## Remote Terraform state

The Terraform configuration uses an Azure Blob Storage backend. The backend
block is intentionally empty in [`provider.tf`](./provider.tf); its values are
supplied during `terraform init` so state storage details do not need to be
committed to the repository.

Create the state resource group, storage account, and blob container with
[`tf-create-backend.sh`](./tf-create-backend.sh). Before running it, set these
values at the top of the script:

- `STATE_RG`: resource group for Terraform state
- `STATE_LOCATION`: Azure region
- `STATE_ACCOUNT`: globally unique storage account name
- `STATE_CONTAINER`: blob container name
- `STATE_KEY`: state blob name, such as `raas.tfstate`
- `GHA_OBJECT_ID`: Microsoft Entra object ID of the GitHub Actions service
  principal

Run the script from the Terraform directory after authenticating with Azure:

```sh
az login
cd terraform
bash tf-create-backend.sh
```

The script grants the GitHub Actions principal `Storage Blob Data Contributor`
on the storage account. If a local `terraform.tfstate` exists, it also runs
`terraform init -migrate-state` to move that state into the new backend.
Review the migration prompt and verify the remote state before deleting any
local state files.

For CI, configure these GitHub repository variables:

- `TF_STATE_RESOURCE_GROUP`
- `TF_STATE_STORAGE_ACCOUNT`
- `TF_STATE_CONTAINER`
- `TF_STATE_KEY`

The Terraform workflow initializes the backend with Azure AD authentication
(`use_azuread_auth=true`) before planning or applying. The workflow's
`AZURE_CREDENTIALS` identity must have access to the state container as well as
the infrastructure resources.

## Create the infrastructure

Log in to Azure and select the target subscription:

```sh
az login
az account set --subscription "<subscription-id>"
```

Initialize Terraform and review the plan:

```sh
terraform init
terraform plan
```

Apply only after reviewing the plan:

```sh
terraform apply
```

Terraform creates the Foundry account, model deployment, private endpoint, and
private DNS resources. No `terraform import` step is required for a new
environment.

If using the GitHub Actions workflow, configure the remote Azure Storage
backend variables and `AZURE_CREDENTIALS` secret, then run the workflow with
`apply` enabled after reviewing the generated plan.

## Outputs and application deployment

After applying, retrieve the backend identity client ID:

```sh
terraform output app_identity_client_id
```

Set that value as the GitHub repository variable `AZURE_APP_CLIENT_ID`. Also
configure the Helm workflow variables and secrets described in
[`../helm/raas/README.md`](../helm/raas/README.md), including
`AZURE_OPENAI_ENDPOINT`.

The application must be deployed after Terraform has created the cluster,
namespace permissions, Foundry account, model deployment, and Workload
Identity configuration. The backend then authenticates to Foundry using its
federated user-assigned identity rather than a stored API key.
