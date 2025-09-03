---
title: Account Setup
date: 2024-11-03
description: >
  Azure - New Account Setup with Critical Security Measures Applied as First Priority.
weight: 1
categories: [Setup, New Account]
tags: [Azure]
---

## Setup New Azure Account
 
Creating an Azure account can often start with a [free tier offering](https://azure.microsoft.com/free/). Similar to other cloud providers, establishing robust security measures from the beginning is crucial to protect your Azure resources, data, and identities from unauthorized access and potential threats. We will follow Azure's recommended practices for a secure initial setup.

### Azure Resource Groups

Azure resource groups are a fundamental concept in Azure to organize resource into specific use case groups for cost and monitoring purposes. Before being able to create an resources in Azure, a [Resource Group](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal) must be created.
 
### Microsoft Entra ID (Azure AD) Foundation
[Microsoft Entra](https://learn.microsoft.com/en-ca/entra/fundamentals/whatis) is Microsoft's cloud-based identity and access management service, more recently know as Microsoft Entra. It's the backbone for managing users, groups, and access to Azure resources. When you sign up for Azure, an Azure AD tenant is typically created for you.

#### MFA Authentication

MFA is a critical security layer to enforce ideally for all users. You can enable MFA through 
* [Security Defaults](https://docs.microsoft.com/azure/active-directory/fundamentals/concept-fundamentals-security-defaults) (a baseline for all users)
*  Configure more granular policies using [Azure AD Conditional Access](https://learn.microsoft.com/en-us/entra/identity/authentication/tutorial-enable-azure-mfa#configure-the-conditions-for-multifactor-authentication)

#### New User

Before creating resources, avoid using the administrative user account created by default when registering a new [Azure Subscription](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions). Create a new user to provide Role-Based Access Control (RBAC) and MFA authentication. Use the [External Invite](https://learn.microsoft.com/en-us/entra/fundamentals/how-to-create-delete-users#invite-an-external-user) mechanism to invite a new user into the Azure AD Tenant.

#### New AD Group & Role Based Access Control (RBAC)

Create [Microsoft Entra ID Groups](https://learn.microsoft.com/en-us/entra/fundamentals/how-to-manage-groups) to apply permissions and access controls across multiple users. Add the newly create user following the [Add Members](https://learn.microsoft.com/en-us/entra/fundamentals/how-to-manage-groups#add-members-or-owners-of-a-group) guide. Grant permissions to the newly created group using Azure RBAC [Role Assignment](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview#role-assignments). Assign the `Reader` role.

#### Conditional Access Policies

We can harden security when accessing Azure subscription/ tenant by utilizing [Conditional Access Policies](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policies) by restricting access to specific IPs. Before defining the policy, create a [Named Location](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-assignment-network#how-are-these-locations-defined) to name and refer to the trusted IP range.

### CLI Access for Users/Developers

To authenticate the Azure CLI and allow users to manage Azure resources from their command line, the primary method is interactive login. This process leverages Microsoft Entra ID for authentication, including any configured MFA or Conditional Access policies.

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
  # Login to Azure (this will open a browser for authentication)
  az login

  # Set the default Subscription and Resource Group
  az account set --subscription [INSERT SUBSCRIPTION ID]
  az configure --defaults group=[INSERT RESOURCE GROUP NAME]

  # [Prefered] If you have a User-Assigned Managed Identity
  # Sign-in to Azure with the User-Assigned Managed Identity
  az login --identity --client-id [INSERT CLIENT ID]
  {{< /tab >}}
{{< /tabpane >}}

## Service Principal (Programmatic Access)

To enable programmatic access to Azure resources, you can create a Service Principal using either:

* Enterprise Application (Service Principal) used by applications, services, or automation tools to access specific Azure resources programmatically.
* User-Assigned Managed Identity, an identity created as a standalone Azure resource, which can be assigned to one or more Azure resources. It is managed by Azure and does not require you to handle credentials. Recommended for scenarios where you want to share an identity across multiple resources.
* System-Assigned Managed Identity, an identity enabled directly on an Azure resource (such as a VM, App Service, or Function). The lifecycle of this identity is tied to the resource, and credentials are managed automatically by Azure. Use this when the identity is only needed for a single resource.


For most automation and service-to-service scenarios, Azure recommends using Managed Identities and short-term credentials. These credentials are generated dynamically and do not require you to manage or rotate long-term secrets.

### Enterprise Application

#### Programmatic Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
export AZURE_RESOURCE_GROUP=example-rg
export AZURE_LOCATION="eastus" # Choose your desired Azure region

# Create a new Azure Resource Group
az group create \
  --name example-rg \
  --location $AZURE_LOCATION

# Create a new Microsoft Entra AD Group
az ad group create \
  --display-name example-group \
  --mail-nickname ex-group

# Create a new Service Principal (Application Registration)
az ad sp create-for-rbac \
  --name example-sp \
  --skip-assignment

# Assign the "Reader" role to the AD Group for the Resource Group
az role assignment create \
  --assignee "example-group" \
  --role Reader \
  --resource-group example-rg
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Remove the Reader role assignment from the AD Group
az role assignment delete \
  --assignee "example-group" \
  --role Reader \
  --resource-group example-rg

# Delete the Service Principal
az ad sp delete --id example-sp

# Delete the AD Group
az ad group delete --group "example-group"

# Delete the Resource Group (removes all resources within it!)
az group delete --name example-rg
  {{< /tab >}}
  {{< tab header="Terraform" lang="HCL" >}}
##---------------------------------------------------------------------------------------------------------------------
## AZUREAD PROVIDER
##
## Azure Active Directory (AzureAD) provider authenticated with CLI.
##---------------------------------------------------------------------------------------------------------------------
provider "azuread" {
  alias = "tokengen"
}

##---------------------------------------------------------------------------------------------------------------------
## AZURRM PROVIDER
##
## Azure Resource Manager (Azurerm) provider authenticated with CLI.
##---------------------------------------------------------------------------------------------------------------------
provider "azurerm" {
  alias = "tokengen"
  features {}
}

##---------------------------------------------------------------------------------------------------------------------
## AZURE SERVICE ACCOUNT MODULE
##
## This module provisions an Azure service account along with associated roles and security groups.
##
## Parameters:
## - `application_display_name`: The display name of the Azure application.
## - `role_name`: The name of the role for the Azure service account.
## - `security_group_name`: The name of the security group.
##
## Providers:
## - `azuread.tokengen`: Alias for the Azure AD provider for generating tokens.
## - `azurerm.tokengen`: Alias for the Azure Resource Manager (Azurerm) provider for generating tokens.
##---------------------------------------------------------------------------------------------------------------------
module "azure_service_account" {
  source  = "github.com/sim-parables/terraform-azure-service-account"

  application_display_name = "example-service-account"
  role_name                = "example-service-account-role"
  security_group_name      = "example-group"

  providers = {
    azuread.tokengen = azuread.tokengen
    azurerm.tokengen = azurerm.tokengen
  }
}
  {{< /tab >}}
{{< /tabpane >}}

#### Console

For applications, services, or automation tools that need to access or modify Azure resources without direct user interaction, use Service Principals. Create a new [Service Principal](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#register-an-application-with-microsoft-entra-id-and-create-a-service-principal) can be used for applications, services or automation tools which need to access or modify Azure resources without direct user interaction. However, SPs require credebtials and token management for authentication which can pose a security risk if handled incorrectly.

### User-Assigned Managed Identities

#### Programmatic Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
# Create a User-Assigned Managed Identity
az identity create \
  --name example-sp-managed-identity \
  --resource-group example-rg \
  --location useast

# (Optional) Assign a role to the Managed Identity for a resource group
az role assignment create \
  --assignee "example-group" \
  --role Reader \
  --resource-group example-rg
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Remove the Reader role assignment from the Managed Identity (if assigned)
az role assignment delete \
  --assignee "example-group" \
  --role Reader \
  --resource-group example-rg

# Delete the User-Assigned Managed Identity
az identity delete \
  --name example-sp-managed-identity \
  --resource-group example-rg
  {{< /tab >}}
{{< /tabpane >}}

#### Console

Managed Identities are [Workload Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) which manage Service Principal and authentication without needing to managed secrets and credentials independently. Create a [User-Assigned Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity) in place of a Service Principal for the same use cases, and apply necessary roles for least-priviledge access. Finalize the configuration of the User-Assigned Managed Identity by [Managing Access](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity) to restrict use to specific users or groups.

