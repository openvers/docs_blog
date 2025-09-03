---
title: Terraform Backend
date: 2024-11-13
description: >
  GCP - Configuring Google Cloud Storage (GCS) as a remote backend for Terraform state, an alternative to Terraform Cloud workspaces.
weight: 2
categories: [Setup]
tags: [GCP, Terraform]
---

# Terraform Backend with Google Cloud Storage

Using Google Cloud Storage (GCS) as the backend for Terraform state is a robust and common practice for managing infrastructure as code on GCP. By storing Terraform state in a GCS bucket, teams can benefit from Google Cloud's strong security features, such as encryption at rest and in transit, fine-grained access control with IAM, and object versioning for state history and recovery. GCS provides a highly durable and available storage solution, ensuring that Terraform state is consistently accessible. This setup allows teams to use IAM Service Accounts and impersonation to manage access to the state files, adhering to the principle of least privilege, similar to using IAM Roles in other cloud environments. Furthermore, it integrates well with other GCP services and CI/CD pipelines for automated and monitored Terraform workflows.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir -p ~/development/test/example_gcp_tf_backend
cd ~/development/test/example_gcp_tf_backend

# Configure GCloud CLI Application-Default-Credentials
# https://cloud.google.com/docs/authentication/application-default-credentials
gcloud auth application-default login

# Setting environment variables
export GCP_PROJECT_ID=$(gcloud config get-value project) # Or set manually: "your-gcp-project-id"
export GCP_REGION="us-central1" # Choose your preferred region
export TERRAFORM_GCS_BUCKET_NAME="${GCP_PROJECT_ID}-tfstate-$(openssl rand -hex 4)" # Must be globally unique
export TERRAFORM_GCS_PREFIX="sandbox/terraform.tfstate"

# Grant StorageAdmin Role to Service Account
gsutil iam ch serviceAccount:$GCP_SA_EMAIL:objectAdmin gs://${TERRAFORM_GCS_BUCKET_NAME}

# Create a GCS Bucket for Terraform State
# https://cloud.google.com/storage/docs/creating-buckets#gsutil
gsutil mb -p $GCP_PROJECT_ID -l $GCP_REGION gs://$TERRAFORM_GCS_BUCKET_NAME

# Enable Object Versioning on the Bucket (Crucial for state history and preventing accidental data loss)
# https://cloud.google.com/storage/docs/object-versioning#gsutil
gsutil versioning set on gs://$TERRAFORM_GCS_BUCKET_NAME

# Configure Terraform
cat <<EOT > backend.conf
bucket                      = "$TERRAFORM_GCS_BUCKET_NAME"
prefix                      = "$TERRAFORM_GCS_PREFIX"
EOT

cat <<EOT > terraform.tfvars
gcp_project_id = "$GCP_PROJECT_ID"
gcp_region     = "$GCP_REGION"
EOT

cat <<EOT > variables.tf
variable "gcp_project_id" {
  type        = string
  description = "GCP Project ID for resource deployment"
}

variable "gcp_region" {
  type        = string
  description = "GCP Region for resource deployment"
}
EOT

cat <<EOT > main.tf
/* GCS managed terraform state - configuration is in backend.conf */
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0" # Specify a version constraint
    }
  }
  backend "gcs" {} // Configuration will be loaded from backend.conf
}

/* Google Cloud Provider Configuration */
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
  // Terraform will use Application Default Credentials (ADC) of the user running it.
  // For CI/CD or specific service account usage for resource management,
  // you can configure 'credentials = file("path/to/sa-key.json")' or use Workload Identity Federation.
}
EOT

# Test Terraform
terraform init -backend-config=backend.conf
terraform plan
# terraform apply --auto-approve # (Optional: Add a dummy resource to main.tf to test apply)
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Ensure environment variables are set from the Create step

# Tear Down Terraform managed resources (if any were created by 'terraform apply')
cd ~/development/test/example_gcp_tf_backend
terraform destroy --auto-approve

# Delete GCS Bucket and all its objects (including versions)
# The '-m' flag enables multi-threaded/multi-processing operations.
# The '-r' flag enables recursive deletion.
gsutil -m rm -r gs://$TERRAFORM_GCS_BUCKET_NAME

# Clean up local Terraform files
rm -f backend.conf main.tf variables.tf terraform.tfvars terraform.tfstate*
rm -rf .terraform .terraform.lock.hcl
  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

### Create GCS Bucket

Create a [GCS Bucket](https://cloud.google.com/storage/docs/creating-buckets) for the terraform `tfstate`. This will reduce the chance of sensitive data being committed to source. Configure the Bucket to have **Standard Storage Classification**, and **Uniform Access Control**. Enabling [GCS Bucket Versioning](https://cloud.google.com/storage/docs/using-object-versioning) is critical for state management - to allow tracking of changes and enable recovery/ roll-back functionality.

### Grant Service Account Permissions to GCS Bucket

Configure the newly created GCS Bucket for terraform `tfstate` to grant Service Account access to read/write following the [Bucket IAM Policies](https://cloud.google.com/storage/docs/access-control/using-iam-permissions) guide. Ensure to assign the Service Account with **Storage Object Admin** role.

### Configure Terraform Backend
Create a `backend.conf` file to configure [GCS Terraform Backend](https://developer.hashicorp.com/terraform/language/backend/gcs). Input the following contents into the file:

```hcl
bucket = "[INSERT_GCS_BUCKET_NAME]"
prefix = "[INSERT_GCS_BUCKET_PREFIX]"
```

Now Terraform will be configured to store its state in GCS. Also, we've omitted configuration `impersonate_service_account`, as the [Google Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials) (ADC) will be automatically impersonating our Service Account when we configure `gcloud cli` impersonation.

```bash
# From New Account Setup
# gcloud config set auth/impersonate_service_account $GCP_SA_EMAIL

gcloud auth application-default login

terraform init -backend-config=backend.conf
terraform plan
terraform apply --auto-approve
```