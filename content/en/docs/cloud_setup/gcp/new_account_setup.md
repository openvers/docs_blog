---
title: Account Setup
date: 2024-11-03
description: >
  GCP - New Account Setup with Critical Security Measures Applied as First Priority.
weight: 1
categories: [Setup, New Account]
tags: [GCP]
---

## Setup New Account

Creating a Google Cloud Platform (GCP) presence starts with a Google Account. If you don't have one, you can create one for free. You'll then create a "Project" within GCP to organize your resources. Billing is managed through a "Billing Account" linked to your projects. More details can be found at the [GCP Get Started page](https://cloud.google.com/docs/get-started).
It's critical to implement robust security measures from the outset to protect your GCP resources and data from unauthorized access and potential breaches. We will follow Google's recommended security best practices.

### GCP Organization Setup (Recommended for Businesses)
If you are using Google Workspace or Cloud Identity, setting up a GCP Organization is highly recommended. An Organization is the root node of the Google Cloud resource hierarchy and provides centralized control over all your GCP resources, projects, and billing. It allows you to define organization-wide policies and manage IAM permissions effectively. However, setting up a GCP organization with free tier Workspaces or Cloud Identity requires at minium a paid domain with email access - **which is out-of-scope for our requirements**.

 * Learn more about [Creating and Managing Organizations](https://cloud.google.com/resource-manager/docs/creating-managing-organization). This is a foundational step for robust governance.
 * Visit [Sign-Up for Google Workspace](https://support.google.com/a/answer/6043576?hl=en&ref_topic=4388346&sjid=6365213940035972062-NC) to create a new Google Workspace, or [Sign-Up for Cloud Identity](https://support.google.com/cloudidentity/answer/7389973?hl=en) for Cloud Identity to register users with limited access to Google Services.

GCP uses Google Accounts for authentication. For managing users, especially in a team or enterprise setting, it's best to use Google Workspace or Cloud Identity. These services act as your identity provider.

*   **Create Users/Groups**: [Manage your users and create groups](https://cloud.google.com/iam/docs/groups-in-cloud-console) (e.g., `project-admins`, `project-developers`) within your Google Workspace or Cloud Identity admin console. Managing users in Cloud Identity.
*   **Enforce Multi-Factor Authentication (MFA/2SV)**: Crucially, enforce 2-Step Verification (Google's term for MFA) for all Google Accounts that will access GCP. This is configured at the Google Account level or enforced via Google Workspace/Cloud Identity policies. Deploy 2-Step Verification.
*   **Grant IAM Roles**: In GCP, you [Grant IAM roles](https://cloud.google.com/iam/docs/grant-role-console) to principals (users, groups, service accounts) on specific resources (Organization, Folders, Projects). Start by granting roles to your Google Groups. For example, grant the `roles/owner` or more specific administrative roles (e.g., `roles/resourcemanager.projectCreator`, `roles/billing.admin`) to your `project-admins` group at the Organization or Folder level. Follow the principle of least privilege. Understanding IAM Roles.
*   **Accessing the GCP Console**: Users will log in to the [Google Cloud Console](https://cloud.google.com) using their Google Account credentials.
### GCP Projects

A GCP Project is the base-level organizing entity for everything you do in Google Cloud. Think of it as a container for all your cloud resources. All GCP resources you create, such as Virtual Machines (Compute Engine instances), Cloud Storage buckets, BigQuery datasets, Cloud SQL instances, IAM Roles and Security Permissions, etc., must belong to a project. We'll need to [Create a GCP Project](https://cloud.google.com/resource-manager/docs/creating-managing-projects) in order to finished the setup steps.

Before continuing, we need to enable the following APIs
* Enable [Cloud Resource Manager API](https://console.developers.google.com/apis/library/cloudresourcemanager.googleapis.com)
* Enable [Identity And Access Management (IAM) API](https://console.cloud.google.com/apis/library/iam.googleapis.com)


### CLI Access for Users

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
  # Initialize gcloud CLI and login with your Google Account
  # This command will open a browser window for authentication.
  # It will also prompt you to choose or create a default project and region/zone.
  gcloud init

  # Alternatively, to just log in (if gcloud is already initialized):
  gcloud auth login

  # To list authenticated accounts:
  gcloud auth list

  # To set your default project:
  gcloud config set project *[YOUR_PROJECT_ID]*
  {{< /tab >}}
{{< /tabpane >}}

## Service Account Setup (Programmatic Access)
For applications, scripts, or CI/CD pipelines that need to interact with GCP APIs programmatically, you should use Service Accounts. Avoid using user credentials for programmatic access.

### Programmatic Steps

You can perform service account setup and permissions assignment using the Google Cloud CLI (`gcloud`). The following steps mirror the console steps above:

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
export GCP_PROJECT_ID=$(gcloud config get-value project)

# Create a new service account
# https://cloud.google.com/iam/docs/creating-managing-service-accounts#gcloud

gcloud iam service-accounts create example-service-account \
  --description="Example Service Account for automation" \
  --display-name="Example Service Account"

# Grant roles to the service account
# https://cloud.google.com/iam/docs/granting-roles-to-service-accounts#gcloud

gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# (Optional) Grant additional roles as needed
# Example: Security Admin
# gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
#   --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
#   --role="roles/iam.securityAdmin"

# Impersonate the service account for short-lived credentials
# https://cloud.google.com/docs/authentication/use-service-account-impersonation#gcloud

gcloud config set auth/impersonate_service_account example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
export GCP_PROJECT_ID=$(gcloud config get-value project)

# Remove IAM policy bindings for the service account
# (Repeat for each role previously assigned)
gcloud projects remove-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects remove-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# (Optional) Remove any additional roles as needed
# gcloud projects remove-iam-policy-binding $GCP_PROJECT_ID \
#   --member="serviceAccount:example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
#   --role="roles/iam.securityAdmin"

# Delete the service account
gcloud iam service-accounts delete \
  example-service-account@${GCP_PROJECT_ID}.iam.gserviceaccount.com
  {{< /tab >}}
  {{< tab header="Terraform" lang="HCL" >}}
## ---------------------------------------------------------------------------------------------------------------------
## GCP PROVIDER
##
## Configures the GCP provider with OIDC Connect via ENV Variables.
## ---------------------------------------------------------------------------------------------------------------------
provider "google" {
  alias = "tokengen"
}

##---------------------------------------------------------------------------------------------------------------------
## GCP SERVICE ACCOUNT MODULE
##
## This module provisions a GCP service account along with associated roles and security groups.
##
## Parameters:
## - `IMPERSONATE_SERVICE_ACCOUNT_EMAIL`: Existing GCP service account email to impersonate for new SA creation.
## - `new_service_account_name`: New service account name.\\
## - `role_list`: List of roles to assign to the new service account.
##
## Providers:
## - `google.tokengen`: Alias for the GCP provider for generating service accounts.
##---------------------------------------------------------------------------------------------------------------------
module "gcp_service_account" {
  source = "../"

  IMPERSONATE_SERVICE_ACCOUNT_EMAIL = var.IMPERSONATE_SERVICE_ACCOUNT_EMAIL
  new_service_account_name          = "example-service-account"

  providers = {
    google.tokengen = google.tokengen
  }
}
  {{< /tab >}}
{{< /tabpane >}}

For more details, see the [gcloud CLI documentation](https://cloud.google.com/sdk/gcloud/reference/iam/service-accounts/).

### Console
Create a [New Service Account](https://cloud.google.com/iam/docs/service-accounts-create) in your GCP project, and give it a descriptive name and ID. Once created, make sure to [Grant Principal Access](https://cloud.google.com/iam/docs/manage-access-service-accounts#grant-single-role) to the Service Account using your project account with the following roles:

 * Service Account Token Creator
 * Service Account User

This will allow users to authenticate and interact with GCP using the Service Account with controlled permissions.

#### Grant Permissions (IAM Roles & Conditions)
Grant the necessary IAM roles to this service account on the specific resources it needs to access follow the principle of least privilege. Under **Permissions**, and selecting **Manage Access** in the Service Account view, is where we will be able to add IAM Roles. For our example, we'll assign the following roles to help manage later steps:

* Security Admin
* Service Account Admin

More discovery and evaluation is needed on hardening security with IP restrictions using [IAM Conditions](https://cloud.google.com/iam/docs/conditions-overview) and [Condition Attributes](https://cloud.google.com/iam/docs/conditions-attribute-reference#request). Currently, only organizations are able to create [Access-Context-Managers](https://cloud.google.com/access-context-manager/docs/create-custom-access-level) to manage IAM Condition level IP restrictions.

It's encourage to [Disable Service Account Key Creation](https://cloud.google.com/resource-manager/docs/organization-policy/restricting-service-accounts#disable_service_account_key_creation) with Organizational IAM Policies, but this is only available to GCP projects belonging to an Organization.

#### Setup Short-Lived Keys with Service Account Impersonation
[Service Account Impersonation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) in GCP allows a principal (like a user or another service account) to temporarily assume the identity and permissions of a specific service account. This is achieved by granting the principal the "Service Account Token Creator" role on the target service account.

Instead of directly using the target service account's long-lived keys, the principal requests short-lived credentials for that service account, effectively acting as that service account for a limited time and with its defined permissions. This is a security best practice as it avoids the need to distribute service account keys and allows for more granular and temporary access control.

