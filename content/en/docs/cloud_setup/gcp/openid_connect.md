---
title: OpenID Connect
date: 2024-11-03
description: >
  GCP - Modernized Security for GitOps & CI/CD with GitHub Actions using Workload Identity Federation.
weight: 3
categories: [Setup, OpenID Connect]
tags: [GCP, GitHub]
---

# OpenID Connect (OIDC) with GitHub Actions and GCP Workload Identity Federation

Google Cloud's [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) allows identities from external OpenID Connect (OIDC) providers, like GitHub Actions, to impersonate Google Cloud service accounts. This enables your GitHub Actions workflows to securely access GCP resources without needing to manage or store long-lived service account keys in GitHub. Instead, workflows obtain short-lived access tokens, enhancing security and simplifying credential management for CI/CD pipelines.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir -p ~/development/test/example_gcp_oidc
cd ~/development/test/example_gcp_oidc
  
## 0. Prerequisites & Configuration
# Ensure gcloud CLI is installed, authenticated, and configured with a default project.
# Ensure GitHub CLI (gh) is installed and authenticated if using it to get repo details.

export GCP_PROJECT_ID=$(gcloud config get-value project)
export GCP_PROJECT_NUMBER=$(gcloud projects describe $GCP_PROJECT_ID --format='value(projectNumber)')
export GH_OWNER="your-github-username-or-org" # Replace with your GitHub username or organization
export GH_REPO="your-github-repo-name"    # Replace with your GitHub repository name
export GCP_USER_EMAIL=$(gcloud config get-value account) # Your GCP user email, for optional SA impersonation

export POOL_ID="github-pool-$(openssl rand -hex 2)"
export PROVIDER_ID="github-provider-$(openssl rand -hex 2)"

## 1. Grant Workload Identity Pool Permissions to Service Account
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:$GCP_SA_EMAIL" \
  --role="roles/iam.workloadIdentityPoolAdmin"


## 1. Create a Workload Identity Pool
# https://cloud.google.com/iam/docs/reference/rest/v1/projects.locations.workloadIdentityPools/create
gcloud iam workload-identity-pools create $POOL_ID \
  --project=$GCP_PROJECT_ID \
  --location="global" \
  --display-name="GitHub Actions Pool" \
  --description="Pool for GitHub Actions OIDC federation"

## 2. Create a Workload Identity Provider in the Pool for GitHub
# https://cloud.google.com/iam/docs/reference/rest/v1/projects.locations.workloadIdentityPools.providers/create
gcloud iam workload-identity-pools providers create-oidc $PROVIDER_ID \
  --project=$GCP_PROJECT_ID \
  --workload-identity-pool=$POOL_ID \
  --location="global" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --allowed-audiences="https://iam.googleapis.com/projects/${GCP_PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/providers/${PROVIDER_ID}" \
  --display-name="GitHub OIDC Provider" \
  --description="Provider for GitHub Actions" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository == '${GH_OWNER}/${GH_REPO}' && assertion.ref == 'refs/heads/main'"
# The --attribute-condition above restricts this provider to only issue credentials for tokens from the specified repo and main branch.
# You can make it broader (e.g., only `assertion.repository_owner == '${GH_OWNER}'`) and control access via IAM conditions on the service account binding.

## 3. Create a dedicated OIDC SA & Grant necessary permissions on GCP resources
# Adjust the role (e.g., roles/iam.securityReviewer, roles/run.invoker, roles/editor) based on what your workflow needs to do.
export GCP_POOL_SA_ID=${POOL_ID}-sa
export GCP_POOL_SA_EMAIL=${GCP_POOL_SA_ID}@${GCP_PROJECT_ID}.iam.gserviceaccount.com
gcloud iam service-accounts create $GCP_POOL_SA_ID
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:${GCP_POOL_SA_EMAIL}" \
  --role="roles/storage.admin"

## 4. Allow GitHub Actions (via the Workload Identity Provider) to impersonate the Service Account
# This binds the GitHub OIDC subject (filtered by the provider's attribute-condition) to the SA.
# The subject from GitHub OIDC token for a specific repo and ref is `repo:OWNER/REPO_NAME:ref:refs/heads/main`
GH_SUBJECT_ASSERTION="repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/main"

gcloud iam service-accounts add-iam-policy-binding $GCP_POOL_SA_EMAIL \
  --project=$GCP_PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principal://iam.googleapis.com/projects/${GCP_PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/subject/${GH_SUBJECT_ASSERTION}"

## 5. (Optional) Allow your GCP user to impersonate the Service Account for testing/local development
gcloud iam service-accounts add-iam-policy-binding $GCP_POOL_SA_EMAIL \
  --project=$GCP_PROJECT_ID \
  --role="roles/iam.serviceAccountTokenCreator" \
  --member="user:${GCP_USER_EMAIL}"

  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Ensure environment variables from the Create step are set, or re-define them:
# 1. (Optional) Remove GCP user's permission to impersonate the Service Account
gcloud iam service-accounts remove-iam-policy-binding $GCP_POOL_SA_EMAIL \
  --project=$GCP_PROJECT_ID \
  --role="roles/iam.serviceAccountTokenCreator" \
  --member="user:${GCP_USER_EMAIL}"

# 2. Remove GitHub Actions' permission to impersonate the Service Account
GH_SUBJECT_ASSERTION="repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/main" # Reconstruct if not set
gcloud iam service-accounts remove-iam-policy-binding $GCP_POOL_SA_EMAIL \
  --project=$GCP_PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principal://iam.googleapis.com/projects/${GCP_PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/subject/${GH_SUBJECT_ASSERTION}"

# 3. Remove roles granted to the Service Account (example: roles/iam.securityReviewer)
gcloud projects remove-iam-policy-binding $GCP_PROJECT_ID \
  --member="serviceAccount:${GCP_POOL_SA_EMAIL}" \
  --role="roles/storage.admin"

# 4. Delete the Service Account
gcloud iam service-accounts delete $GCP_POOL_SA_EMAIL \
  --project=$GCP_PROJECT_ID

# 5. Delete the Workload Identity Provider
gcloud iam workload-identity-pools providers delete $PROVIDER_ID \
  --project=$GCP_PROJECT_ID \
  --workload-identity-pool=$POOL_ID \
  --location="global"

# 6. Delete the Workload Identity Pool
gcloud iam workload-identity-pools delete $POOL_ID \
  --project=$GCP_PROJECT_ID \
  --location="global"

  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

### 1. Create Workload Identity Pool

Start by creating a new [Workload Identity Pool](https://cloud.google.com/iam/docs/manage-workload-identity-pools-providers) - provide a name like `github-actions-pool`. The Workload Identity Pool (WIP) will centralize the collection of possible OIDC providers we can configure to provide least-priviledged access into our GCP Project.

### 2. Add a Provider to the Pool for GitHub Actions

After creating our WIP, we can now [Manage Providers](https://cloud.google.com/iam/docs/manage-workload-identity-pools-providers#manage-providers). Setup instructions for [OIDC between GCP and Github](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform#adding-a-google-cloud-workload-identity-provider) are sparse, but more cross-platform details can be found in the [Security Hardening Guide](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#configuring-the-subject-in-your-cloud-provider). Use the following details to setup Github OIDC provider in our WIP.

* **Issuer (URL)** Enter `https://token.actions.githubusercontent.com`.
* **Attribute Mapping** details/ maps the [OIDC Token Claims](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token) for CEL expression filtering
    *   `google.subject`: `assertion.sub`
    *   `attribute.actor`: `assertion.actor`
    *   `attribute.aud`: `assertion.aud`
    *   `attribute.repository`: `assertion.repository` (e.g., `your-owner/your-repo`)
    *   `attribute.repository_owner`: `assertion.repository_owner`
    *   `attribute.ref`: `assertion.ref` (e.g., `refs/heads/main`)
    *   `attribute.environment`: `assertion.environment` (if you use GitHub environments)
* **Attribute Condition (Optional but Recommended)**:
    *   This CEL expression filters which GitHub tokens are allowed by this provider.
    *   Example for a specific repository and its main branch:
        `assertion.repository == 'YOUR_GITHUB_OWNER/YOUR_GITHUB_REPO' && assertion.ref == 'refs/heads/main'`
    *   Example for any repository under a specific owner:
        `assertion.repository_owner == 'YOUR_GITHUB_OWNER'`

### 3. Create Service Account(s) & Grant Workload Identity User Role
You'll need one or more [New Service Account](https://cloud.google.com/iam/docs/service-accounts-create) that your GitHub Actions workflows will impersonate. Configure the Workload Identity Pool (i.e., specific GitHub Actions workflows) with [WIP Access Management](https://cloud.google.com/iam/docs/workload-identity-federation#access_management) to impersonate the service account(s). [Grant Principal Access](https://cloud.google.com/iam/docs/manage-access-service-accounts#grant-single-role) to the Service Account with either of the following Principals:
* To allow workflows from a specific repository and specific branch (e.g., `main`):
        `principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/subject/repo:GITHUB_OWNER/GITHUB_REPO_NAME:ref:refs/heads/main`
* To allow workflows from a specific repository and specific environment (e.g., `production`):
        `principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/subject/repo:GITHUB_OWNER/GITHUB_REPO_NAME:environment:production`
**Replace `PROJECT_NUMBER`, `POOL_ID`, `PROVIDER_ID`, `GITHUB_OWNER`, `GITHUB_REPO_NAME` with your actual values.**

**Assign roles**: **Workload Identity User** (`roles/iam.workloadIdentityUser`) to allow for Workload Identity Federation impersonation with the Service Account. Lastly, bind the necessary roles with least priviledges to operate. Repeat for other service accounts and GitHub principal combinations as needed. For example:
    *   `github-cicd-read-sa` might be impersonable by `repo:ORG/REPO:ref:refs/heads/*` (any branch).
    *   `github-cicd-deploy-sa` might be impersonable by `repo:ORG/REPO:ref:refs/heads/main` or `repo:ORG/REPO:environment:production`.

