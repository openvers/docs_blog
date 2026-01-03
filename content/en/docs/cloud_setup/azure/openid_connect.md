---
title: OpenID Connect
date: 2024-11-03
description: >
  Azure - Modernized Security for GitOps & CI/CD with GitHub Actions using Microsoft Identity Platform.
weight: 3
categories: [Setup, OpenID Connect]
tags: [Azure, GitHub]
---

# OpenID Connect (OIDC) with GitHub Actions and Azure Workload Identity Federation

Azure's [Workload identity federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation) allows external OpenID Connect (OIDC) providers, like GitHub Actions, to authenticate to Azure Active Directory (Azure AD) and obtain access tokens for Azure resources. This is achieved by configuring a federated identity credential on an Azure AD application registration or user-assigned managed identity. This enables your GitHub Actions workflows to securely access Azure resources without needing to manage or store long-lived secrets (like client secrets or certificates) in GitHub. Instead, workflows exchange their OIDC token for a short-lived Azure AD access token, enhancing security and simplifying credential management for CI/CD pipelines.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir -p ~/development/test/example_azure_oidc
cd ~/development/test/example_azure_oidc

## 0. Prerequisites & Configuration
# Ensure Azure CLI (az) is installed and you are logged in.
# Run `az login` and `az account set --subscription "YOUR_SUBSCRIPTION_ID"` if needed.
# Ensure GitHub CLI (gh) is installed and authenticated if using it to get repo details.

# Azure Configuration
export AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export AZURE_RESOURCE_GROUP_NAME=$(az configure --list-defaults --query "[?name=='group'].value" -o tsv)
export AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)

# GitHub Configuration
export GH_OWNER="your-github-username-or-org" # Replace with your GitHub username or organization
export GH_REPO="your-github-repo-name"    # Replace with your GitHub repository name

# Azure AD Application and Federated Credential Configuration
export AZURE_APP_REG_DISPLAY_NAME="github-${GH_OWNER}-${GH_REPO}-oidc-$(openssl rand -hex 2)"
export AZURE_FEDERATED_CREDENTIAL_NAME="github-federated-credential-$(openssl rand -hex 2)"

# Subject claim for the federated credential (identifies the GitHub Actions workflow)
# This example targets the main branch. Adjust as needed (e.g., for specific tags, pull requests, or environments).
# Common subject formats:
# - Specific branch: repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/main
# - Any branch: repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/*
# - Any tag: repo:${GH_OWNER}/${GH_REPO}:ref:refs/tags/*
# - Pull requests: repo:${GH_OWNER}/${GH_REPO}:pull_request
# - Specific environment: repo:${GH_OWNER}/${GH_REPO}:environment:production
export GH_SUBJECT_CLAIM="repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/main"

## 1. Create an Azure AD Application Registration
# This application will represent your GitHub Actions workflow in Azure AD.
APP_REG_JSON=$(az ad app create --display-name "$AZURE_APP_REG_DISPLAY_NAME" --sign-in-audience AzureADMyOrg)
export AZURE_APP_OBJECT_ID=$(echo $APP_REG_JSON | jq -r '.id')
export AZURE_APP_CLIENT_ID=$(echo $APP_REG_JSON | jq -r '.appId') # Also known as Application (client) ID

## 2. Create a Service Principal for the Application Registration (if not already created)
# A service principal is an instance of an application in a directory.
# It's usually created automatically when the app registration is created.
# We need its Object ID for role assignments.
SP_JSON=$(az ad sp create --id $AZURE_APP_CLIENT_ID) # This command creates if not exists, or returns existing
export AZURE_SERVICE_PRINCIPAL_OBJECT_ID=$(echo $SP_JSON | jq -r '.id')

## 3. Add a Federated Identity Credential to the Application Registration
# This configures the trust relationship between Azure AD and GitHub OIDC.
az ad app federated-credential create \
  --id $AZURE_APP_OBJECT_ID \
  --parameters '{
    "name": "${AZURE_FEDERATED_CREDENTIAL_NAME}",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "${GH_SUBJECT_CLAIM}",
    "description": "Federated credential for GitHub Actions OIDC",
    "audiences": ["api://AzureADTokenExchange"]
  }'

## 4. Grant necessary permissions (Azure RBAC roles) to the Service Principal
# Adjust the role and scope based on what your GitHub Actions workflow needs to do.
# Example: Grant "Storage Blob Data Contributor" on a specific resource group.
az role assignment create \
  --assignee $AZURE_SERVICE_PRINCIPAL_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$AZURE_RESOURCE_GROUP_NAME"
# To grant on the entire subscription: --scope "/subscriptions/$AZURE_SUBSCRIPTION_ID"

  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Ensure environment variables from the Create step are set, or re-define them:

# Retrieve IDs if not set (assuming display name is unique enough for this script)
export AZURE_APP_OBJECT_ID=$(az ad app list --display-name "$AZURE_APP_REG_DISPLAY_NAME" --query "[0].id" -o tsv)
export AZURE_APP_CLIENT_ID=$(az ad app show --id "$AZURE_APP_OBJECT_ID" --query "appId" -o tsv)
export AZURE_SERVICE_PRINCIPAL_OBJECT_ID=$(az ad sp list --display-name "$AZURE_APP_REG_DISPLAY_NAME" --query "[?appId=='$AZURE_APP_CLIENT_ID'].id" -o tsv)

# 1. Remove Azure RBAC role assignments granted to the Service Principal
# You need to know the role and scope that was assigned.
az role assignment delete \
  --assignee $AZURE_SERVICE_PRINCIPAL_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$AZURE_RESOURCE_GROUP_NAME"

# 2. Delete the Federated Identity Credential
# To delete a specific federated credential, you'd ideally use its ID.
# Listing them to find the one to delete if name is known:
CREDENTIAL_ID=$(az ad app federated-credential list --id $AZURE_APP_OBJECT_ID --query "[?name=='$AZURE_FEDERATED_CREDENTIAL_NAME'].id" -o tsv)
az ad app federated-credential delete --id $AZURE_APP_OBJECT_ID --federated-credential-id $CREDENTIAL_ID

# 3. Delete the Azure AD Application Registration
# This will also delete the associated Service Principal.
az ad app delete --id $AZURE_APP_OBJECT_ID

  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

### 1. Create Azure AD Application Registration

Begin this OIDC process by first [Registering a Client Application](https://learn.microsoft.com/en-us/azure/healthcare-apis/register-application) in Microsoft Entra ID. Provide a name like `github-actions-myrepo-oidc`, and note down the **Application (client) ID**.


### 2. Add a Federated Identity Credential

Once the App Registration is created, configure [Workload Identity Federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) to establish [Trust with GitHub Actions OIDC](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust?pivots=identity-wif-apps-methods-azp#github-actions). 

**Example Identities for GitHub Actions in Azure Portal (Subject Identifier in Federated Credential):**
*   To allow workflows from a specific repository and specific branch (e.g., `main`):
    `repo:GITHUB_OWNER/GITHUB_REPO_NAME:ref:refs/heads/main`
*   To allow workflows from a specific repository and specific environment (e.g., `production`):
    `repo:GITHUB_OWNER/GITHUB_REPO_NAME:environment:production`

*Also, a managed identity can be configured to establish trust with Github Actions OIDC. More details [here](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity?pivots=identity-wif-mi-methods-azp)*


### 3. Grant Permissions (Assign Azure RBAC Roles) to the Service Principal

The App Registration has an associated Service Principal. You need to [Grant this Service Principal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-steps) the necessary permissions (Azure RBAC roles) on the Azure resources your GitHub Actions workflow will manage.
Select the appropriate **Role** (e.g., `Contributor`, `Reader`, `Storage Blob Data Contributor`) that provides the least privilege necessary. 

**Assign roles**: Assign the necessary Azure RBAC roles to the Service Principal of your App Registration. For example:
*   `github-cicd-read-sp` might be impersonable by `repo:ORG/REPO:ref:refs/heads/*` (any branch) and have the `Reader` role on a resource group.
*   `github-cicd-deploy-sp` might be impersonable by `repo:ORG/REPO:ref:refs/heads/main` or `repo:ORG/REPO:environment:production` and have the `Contributor` role on a resource group.
