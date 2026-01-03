---
title: Terraform Backend
date: 2024-11-13
description: >
  AWS - Optional Configurations When Not Using Terraform.io Workspace Backends.
weight: 2
categories: [Setup]
tags: [AWS, Terraform]
---

# Terrafrom Backend with AWS

Using AWS as the backend for Terraform state is another versatile option for managing infrastructure as code. By storing Terraform state in an AWS S3 bucket, teams can take advantage of AWS's robust security features, such as encryption at rest and in transit, access controls, and versioning. Additionally, AWS S3 provides a highly durable and available storage solution, ensuring that Terraform state is always accessible and up-to-date. Furthermore, by using AWS as the backend, teams can also leverage AWS IAM to manage access to the Terraform state, allowing for fine-grained control over who can read and write to the state file. This setup also enables teams to use AWS services such as AWS Lambda and AWS CloudWatch to automate and monitor their Terraform workflows, providing a seamless and integrated experience for managing infrastructure as code.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir -p ~/development/test/example_aws_tf_backend
cd ~/development/test/example_aws_tf_backend
  
# Setting
export AWS_TERRAFORM_S3_BUCKET=$(openssl rand -hex 4)-example-tf-state-bucket
export AWS_TERRAFORM_S3_BUCKET_PATH=sandbox/terraform.tfstate
export AWS_TERRAFORM_DYNAMODB_TABLE=sandox_terraform_tfstate

# Login to AWS root account
aws sso login --profile=sso-admin

# Create an S3 Bucket for Terraform State
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api/create-bucket.html
aws s3api create-bucket \
  --acl private \
  --bucket $AWS_TERRAFORM_S3_BUCKET \
  --region us-east-1 \
  --profile=sso-admin
  
# Create a DynamoDB Table for Terraform State
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/dynamodb/create-table.html
aws dynamodb create-table \
  --table-name $AWS_TERRAFORM_DYNAMODB_TABLE \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
  --profile=sso-admin
  
# Get Caller Identity ARN
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sts/get-caller-identity.html
export AWS_TERRAFORM_IAM_USER_ARN=$(aws sts get-caller-identity \
  --query 'Arn' \
  --output text \
  --profile=sso-admin)
  
# Create IAM Policy for Terraform Backend Auth
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
cat <<EOT > aws_sts_sandbox_policy_document.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SandboxTerraformBackendS3BucketPermissions",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::$AWS_TERRAFORM_S3_BUCKET"
    },
    {
      "Sid": "SandboxTerraformBackendS3BlobPermissions",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::$AWS_TERRAFORM_S3_BUCKET/$AWS_TERRAFORM_S3_BUCKET_PATH"
    },
    {
      "Sid": "SandboxTerraformBackendDynamoDBTablePermissions",
      "Effect": "Allow",
      "Action": [
        "dynamodb:DescribeTable",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/$AWS_TERRAFORM_DYNAMODB_TABLE"
    }
  ]
}
EOT
  
# Create IAM Trust Policy for IAM Role
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
cat <<EOT > aws_sts_sandbox_role_trust_policy_document.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SandboxTerraformTrustPolicy",
      "Effect": "Allow",
      "Principal": {
        "AWS": "$AWS_TERRAFORM_IAM_USER_ARN"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOT

# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
export AWS_TERRAFORM_IAM_POLICY_ARN=$(aws iam create-policy \
  --policy-name example-tf-sandbox-policy \
  --policy-document file://aws_sts_sandbox_policy_document.json \
  --query 'Policy.Arn' \
  --output text \
  --profile=sso-admin)
  
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-role.html
export AWS_TERRAFORM_IAM_ROLE_ARN=$(aws iam create-role \
  --role-name example-tf-sandbox-role \
  --assume-role-policy-document file://aws_sts_sandbox_role_trust_policy_document.json \
  --query 'Role.Arn' \
  --output text \
  --profile=sso-admin)
  
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/attach-role-policy.html
aws iam attach-role-policy \
  --policy-arn $AWS_TERRAFORM_IAM_POLICY_ARN \
  --role-name example-tf-sandbox-role \
  --profile=sso-admin

# Configure Terraform
cat <<EOT > backend.conf
bucket                       = "$AWS_TERRAFORM_S3_BUCKET"
key                          = "$AWS_TERRAFORM_S3_BUCKET_PATH"
region                       = "us-east-1"
encrypt                      = true
dynamodb_table               = "$AWS_TERRAFORM_DYNAMODB_TABLE"
profile                      = "sso-admin"
role_arn                     = "$AWS_TERRAFORM_IAM_ROLE_ARN"
assume_role_duration_seconds = 900
EOT

cat <<EOT > terraform.tfvars
aws_cli_profile = "sso-admin"
aws_region      = "us-east-1"
EOT

cat <<EOT > variables.tf
variable "aws_cli_profile" {
  type        = string
  description = "AWS CLI Profile Name"
}

variable "aws_region" {
  type        = string
  description = "AWS Account Region"
}
EOT

cat <<EOT > maint.tf
/* AWS S3 managed terraform state - configurations in backend.conf */
terraform {
  backend "s3" {}
}

/* Proxy AWS Provider */
provider "aws" {
  alias   = "example-sso-cli"
  profile = var.aws_cli_profile
  region  = var.aws_region
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
cd ~/development/test/example_aws_tf_backend
terraform destroy --auto-approve

# Remove IAM Roles & Policies
aws iam detach-role-policy \
  --policy-arn $AWS_TERRAFORM_IAM_POLICY_ARN \
  --role-name example-tf-sandbox-role \
  --profile=sso-admin

aws iam delete-role \
  --role-name example-tf-sandbox-role \
  --profile=sso-admin

aws iam delete-policy \
  --policy-arn $AWS_TERRAFORM_IAM_POLICY_ARN \
  --profile=sso-admin

# Remove DynamoDB
aws dynamodb delete-table \
  --table-name $AWS_TERRAFORM_DYNAMODB_TABLE \
  --profile=sso-admin

# Remove S3 Versioned Objects & Bucket
aws s3api delete-objects \
  --bucket $AWS_TERRAFORM_S3_BUCKET \
  --delete "$(aws s3api list-object-versions \
    --bucket $AWS_TERRAFORM_S3_BUCKET \
    --output=json \
    --query='{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
    --profile=sso-admin)" \
  --profile=sso-admin

aws s3api delete-bucket \
  --bucket $AWS_TERRAFORM_S3_BUCKET \
  --profile=sso-admin
  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

### Create S3 Bucket

Create a Private [S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html) for Terraform `tfstate`. This will reduce the chance of sensitive data being committed to source code.

### Create DynamoDB Table

Create a new [DynamoDB Table](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-1.html) in AWS with a Partition Key named [LockID](https://developer.hashicorp.com/terraform/language/settings/backends/s3#dynamodb-state-locking).

### Create IAM Policy

[Configure Terraform Backend](https://developer.hashicorp.com/terraform/language/settings/backends/s3) to store `tfstate` to S3 by creating a new policy for AWS service principals authorized with Terraform.

```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::[INSERT_BUCKET_NAME]"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::[INSERT_BUCKET_NAME]/[INSERT_PATH_KEY]"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeTable",
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/[INSERT_TABLE_NAME]"
        }
     ]
   }
```

### Create IAM Role with Trust Policy
Create a new [IAM Role with Custom Trust Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-custom.html) to allow the Identity Center user with SSO access to [Assume the Role](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) with the above Policy permissions. Make not of the Role ARN for the next step.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SandboxTerraformTrustPolicy",
      "Effect": "Allow",
      "Principal": {
        "AWS": "[INSERT_IDENTITY_CENTER_USER_ARN]"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Configure Terraform Backend
Utilize [Terraform Partial Configuration](https://developer.hashicorp.com/terraform/language/backend#partial-configuration) which may point to a file with the backend configurations stored in a safer location than hard coding into the `main.tf` file. Write the contents below into a `backend.conf` file in the parent directory (and make sure to include into .gitignore).

```hcl
bucket                       = "[INSERT_S3_BUCKET_NAME]"
key                          = "[INSERT_S3_BUCKET_PATH]"
region                       = "us-east-1"
encrypt                      = true
dynamodb_table               = "[INSERT_DYNAMODB_TABLE_NAME]"
profile                      = "sso-admin"
role_arn                     = "[INSERT_IAM_ROLE_ARN]"
assume_role_duration_seconds = 900
```

Now Terraform will be configured to store all `tfstate` in AWS S3 and DynamoDB.

```bash
terraform init -backend-config=backend.conf
terraform plan
terraform apply --auto-approve
```