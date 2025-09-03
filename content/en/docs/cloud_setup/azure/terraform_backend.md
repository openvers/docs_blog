---
title: Terraform Backend
date: 2024-11-13
description: >
  Azure - Optional Configurations When Not Using Terraform.io Workspace Backends.
weight: 2
categories: [Setup]
tags: [Azure, Terraform]
---

# Terraform Backend with Azure

Using Azure as the backend for Terraform state is a robust and versatile option for managing infrastructure as code. By storing Terraform state in an Azure Blob Storage container, teams can leverage Azure's comprehensive security features, including encryption at rest and in transit, access controls via Azure Active Directory (Azure AD) and Role-Based Access Control (RBAC), and versioning capabilities. Azure Blob Storage offers a highly durable and available storage solution, ensuring that Terraform state is consistently accessible and protected. Furthermore, using Azure for the backend allows teams to utilize Azure AD for fine-grained control over who can read and write to the state file and manage state locking. This setup also integrates well with other Azure services like Azure Functions and Azure Monitor, enabling automation and monitoring of Terraform workflows for a seamless infrastructure management experience.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir -p ~/development/test/example_azure_tf_backend
cd ~/development/test/example_azure_tf_backend
  
# Setting
export AZURE_LOCATION="eastus" # Choose your desired Azure region
export AZURE_CLIENT_ID=$(az identity list | head -n 1)
export RAND_SUFFIX=$(openssl rand -hex 4)
export AZURE_RESOURCE_GROUP_NAME="rg-tfstate-${RAND_SUFFIX}"

# Azure Storage Account names must be 3-24 characters, lowercase letters and numbers, and globally unique.
export AZURE_STORAGE_ACCOUNT_NAME=$(echo "tfstate-${RAND_SUFFIX}" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]//g' | cut -c1-24)
export AZURE_STORAGE_CONTAINER_NAME="tfstate"
export TF_STATE_KEY="sandbox/terraform.tfstate" # Path for the Terraform state file in the container

# Login to Azure
az login --identity --client-id $AZURE_CLIENT_ID

# Get Subscription and Tenant IDs for Terraform configuration
export AZURE_SUBSCRIPTION_ID_FOR_TF=$(az account show --query id -o tsv)
export AZURE_TENANT_ID_FOR_TF=$(az account show --query tenantId -o tsv)

# Create a Resource Group for the Terraform backend
# https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az_group_create
az group create \
  --name "$AZURE_RESOURCE_GROUP_NAME" \
  --location "$AZURE_LOCATION"

az configure --defaults group=$AZURE_RESOURCE_GROUP_NAME

# Create a Storage Account for Terraform State
# https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az_storage_account_create
az storage account create \
  --name "$AZURE_STORAGE_ACCOUNT_NAME" \
  --resource-group "$AZURE_RESOURCE_GROUP_NAME" \
  --location "$AZURE_LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --encryption-services blob \
  --allow-blob-public-access false

# Create a Blob Container for Terraform State
# https://docs.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest#az_storage_container_create
az storage container create \
  --name "$AZURE_STORAGE_CONTAINER_NAME" \
  --account-name "$AZURE_STORAGE_ACCOUNT_NAME"

# Get the Object ID (Principal ID) of the managed identity's service principal using its Client ID.
export MANAGED_IDENTITY_OBJECT_ID=$(az ad sp show --id "$AZURE_CLIENT_ID" --query "id" -o tsv)
# AZURE_SUBSCRIPTION_ID_FOR_TF is already set and can be used for the scope.
export CONTAINER_SCOPE="/subscriptions/$AZURE_SUBSCRIPTION_ID_FOR_TF/resourceGroups/$AZURE_RESOURCE_GROUP_NAME/providers/Microsoft.Storage/storageAccounts/$AZURE_STORAGE_ACCOUNT_NAME/blobServices/default/containers/$AZURE_STORAGE_CONTAINER_NAME"

az role assignment create \
    --role "Storage Blob Data Contributor" \
    --assignee-object-id "$MANAGED_IDENTITY_OBJECT_ID" \
    --assignee-principal-type ServicePrincipal \
    --scope "$CONTAINER_SCOPE"
# Role assignment may take a moment to propagate. Wait for 30 seconds before continuing.


# Configure Terraform
cat <<EOT > backend.conf
resource_group_name  = "$AZURE_RESOURCE_GROUP_NAME"
storage_account_name = "$AZURE_STORAGE_ACCOUNT_NAME"
container_name       = "$AZURE_STORAGE_CONTAINER_NAME"
key                  = "$TF_STATE_KEY"
use_azuread_auth     = true # Authenticate using Azure Active Directory.
# Explicitly use the managed identity (whose client ID is in $AZURE_CLIENT_ID)
# for backend authentication. The `az login --identity` command sets the CLI context,
# and these details ensure Terraform uses that specific identity for the backend.
client_id            = "$AZURE_CLIENT_ID"
subscription_id      = "$AZURE_SUBSCRIPTION_ID_FOR_TF"
tenant_id            = "$AZURE_TENANT_ID_FOR_TF"
EOT

cat <<EOT > terraform.tfvars
azure_subscription_id            = "$AZURE_SUBSCRIPTION_ID_FOR_TF"
azure_tenant_id                  = "$AZURE_TENANT_ID_FOR_TF"
user_assigned_identity_client_id = "$AZURE_CLIENT_ID"
# azure_location                 = "$AZURE_LOCATION" # Example if you have a location variable
# You can define variables for your Azure resources here if needed.
# For example, if your main.tf creates resources in a specific location:
# location = "East US"
EOT

cat <<EOT > variables.tf
variable "user_assigned_identity_client_id" {
  type        = string
  description = "Client ID of the User-Assigned Managed Identity to use for authentication by the provider."
}

variable "azure_subscription_id" {
  type        = string
  description = "Azure Subscription ID to be used by the Azure provider."
}

variable "azure_tenant_id" {
  type        = string
  description = "Azure Tenant ID to be used by the Azure provider."
}

# variable "azure_location" {
#   type        = string
#   description = "The Azure region where resources will be deployed."
#   default     = "East US"
# }
EOT

cat <<EOT > main.tf
/* Azure Blob Storage managed terraform state - configurations in backend.conf */
terraform {
  backend "azurerm" {
    // Backend configuration is provided via the backend.conf file
    // during `terraform init -backend-config=backend.conf`
  }
}

provider "azurerm" {
  features {}
  # Explicitly configure the provider to use the User-Assigned Managed Identity.
  # The values are supplied through terraform.tfvars (sourced from script variables).
  # This ensures Terraform uses the specified managed identity for managing Azure resources.
  use_msi         = true
  client_id       = var.user_assigned_identity_client_id
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
}

# Example: Add a data source to verify provider configuration
data "azurerm_subscription" "current" {}

output "current_azure_subscription_display_name" {
  value = data.azurerm_subscription.current.display_name
}
EOT

# Test Terraform
terraform init -backend-config=backend.conf
terraform plan
terraform apply --auto-approve
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Ensure environment variables are set from the Create step

# Tear Down Terraform
cd ~/development/test/example_azure_tf_backend # Ensure you are in the correct directory
terraform destroy --auto-approve

# Remove the Azure Resource Group containing the Storage Account and Container
# This will delete all resources created by this script for the backend.
az group delete \
  --name "$AZURE_RESOURCE_GROUP_NAME" \
  --yes \
  --no-wait # Use --no-wait to run in background, or remove for synchronous deletion

  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

This section outlines how to set up the Azure backend for Terraform using the Azure portal. You will need to manually create the necessary Azure resources and then configure your Terraform files.

### Create Azure Resources

You'll need to create the following resources in the Azure portal. Make sure to note down their names and other relevant properties (like location) as you create them.

*   **Resource Group:** A container that holds related resources for an Azure solution.
    *   [Learn how to create a Resource Group in the Azure portal](https://learn.microsoft.com/azure/azure-resource-manager/management/manage-resource-groups-portal)
*   **Storage Account:** To store your Terraform state file.
    *   When creating a [Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal), ensure you configure it appropriately for a Terraform backend (e.g., Account kind: `StorageV2`, Performance: `Standard`, Replication: `Locally-redundant storage (LRS)`). It's also recommended to disable public blob access for security. Add a Storage Container for the state file.

### Configure Managed Identity and Permissions

Terraform will authenticate to Azure using a User-Assigned Managed Identity.

*   **Assign 'Storage Blob Data Contributor' Role:** [Grant the Managed Identity]((https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity)) the necessary permissions to read and write to the blob container by assigning the `Storage Blob Data Contributor` role to your User-Assigned Managed Identity.

### Configure Terraform Files

After setting up the Azure resources and permissions, you need to create the following [Terraform Backend Configuration](https://developer.hashicorp.com/terraform/language/backend/azurerm) `backend.conf` with the actual values you obtained from the Azure portal.

```hcl
resource_group_name  = "YOUR_RESOURCE_GROUP_NAME"
storage_account_name = "YOUR_STORAGE_ACCOUNT_NAME"
container_name       = "YOUR_CONTAINER_NAME"
key                  = "sandbox/terraform.tfstate" # Or your desired state file path in the container
use_azuread_auth     = true
client_id            = "YOUR_MANAGED_IDENTITY_CLIENT_ID"
subscription_id      = "YOUR_AZURE_SUBSCRIPTION_ID"
tenant_id            = "YOUR_AZURE_TENANT_ID"
```

Once the Azure resources are ready and your Terraform files are configured with the correct values, navigate to your Terraform project directory in your terminal and run:

```bash
terraform init -backend-config=backend.conf
terraform plan
terraform apply --auto-approve
```