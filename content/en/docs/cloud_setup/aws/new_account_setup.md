---
title: Account Setup
date: 2024-11-03
description: >
  AWS - New Account Setup with Critical Security Measures Applied as First Priority.
weight: 1
categories: [Setup, New Account]
tags: [AWS]
---

## Setup New Account

Creating an AWS account can be done free of charge. More details about registering with AWS can be found at their [New Account Instructions](https://aws.amazon.com/resources/create-account/) web page. We also need to keep in mind securing protective measures against our newly created users to protect our resources and data from possible breaches and mailicous actors, and therefore we follow some elementary guidelines documented by AWS for securing new user setup.

### AWS Organization
Enable AWS Organizations following [Instructions Here](https://docs.aws.amazon.com/singlesignon/latest/userguide/get-set-up-for-idc.html). This does not require additional funding, and is a requirement for the next step.

### IAM Identity Center
Before we begin creating new users, we should enable SSO login into AWS Console for our account. This can be done by [Enabling IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/get-set-up-for-idc.html). 

* [Add IAM Identity User](https://docs.aws.amazon.com/singlesignon/latest/userguide/addusers.html#:~:text=To%20add%20a%20user&text=Choose%20Users.,can't%20be%20changed%20later.) to send invite to your root user email

* Include configuration for [Multi-Account Permissions AWS Accounts](https://docs.aws.amazon.com/singlesignon/latest/userguide/assignusers.html) to assign user to newly created AWS Account. Proceed to include the permission sets with pre-defined **AdministratorAccess**.

* Navigate to Users and send invite via email. Proceed to check invite from email and setup MFA authentication.

### CLI Access for Users/Developers

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
  # Configure & Login to AWS root account
  # https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html
  # 
  # SSO session name (Recommended): my-sso
  # SSO start URL [None]: [COPY/PASTE FROM IAM Identity Center -> Settings -> Identity Source -> AWS access portal URL]
  # SSO region [None]: us-east-1
  # SSO registration scopes [None]: sso:account:access
  # CLI profile name [PermissionSet-AWS_Account]: sso-admin
  aws configure sso

  aws sso login --profile=sso-admin
  
  {{< /tab >}}
{{< /tabpane >}}

Remember! You can always find your IAM Identity Center Access Portal in the CLI after logging in.

## Service Principal (Programmatic Access)

To enable programmatic access to AWS resources, you can create a Service Principal using either:

- An IAM User with programmatic access (requires long-term access keys), or
- An IAM Role, which can be assumed to obtain short-term credentials (no long-term secret required).

For most automation and service-to-service scenarios, AWS recommends using IAM Roles and short-term credentials. These credentials are generated dynamically and do not require you to manage or rotate long-term secrets.

### Programmatic Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}
export AWS_SSO_CURRENT_IDENTITY=$(aws sts get-caller-identity \
  --profile=sso-admin)

export AWS_SSO_INSTANCE=$(aws sso-admin list-instances \
  --profile=sso-admin)

export AWS_SSO_CURRENT_USER=$(echo $AWS_SSO_CURRENT_IDENTITY | \
  jq -r '.UserId' | \
  cut -d':' -f2)

export AWS_SSO_IDENTITY_STORE_ID=$(echo $AWS_SSO_INSTANCE | \
  jq -r '.Instances[0].IdentityStoreId')

export AWS_SSO_CURRENT_PRINCIPAL=$(aws identitystore list-users \
--identity-store-id $AWS_SSO_IDENTITY_STORE_ID \
--query='Users' \
--profile=sso-admin | \
  jq -r --arg user $AWS_SSO_CURRENT_USER 'map(select(.UserName | contains($user)))[0]'
)


# 1. Create a new IAM Identity Center Permission Set
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sso-admin/create-permission-set.html
export AWS_SSO_PERMISSION_SET=$(aws sso-admin create-permission-set \
  --instance-arn $(echo $AWS_SSO_INSTANCE | jq -r '.Instances[0].InstanceArn') \
  --name "ExampleCustomViewerPermissionSet" \
  --description "Example viewer permissions for my team" \
  --session-duration "PT8H" \
  --profile=sso-admin
)

# 2. Attach a Managed Policy
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sso-admin/attach-managed-policy-to-permission-set.html
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn $(echo $AWS_SSO_INSTANCE | jq -r '.Instances[0].InstanceArn') \
  --permission-set-arn $(echo $AWS_SSO_PERMISSION_SET | jq -r '.PermissionSetArn') \
  --managed-policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess \
  --profile=sso-admin

# 3. Assign Account Access and Permissions to User
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sso-admin/create-account-assignment.html
# This is for for example purposes as normally you would not
# assigning yourself permissions as an administrator

aws sso-admin create-account-assignment \
  --instance-arn $(echo $AWS_SSO_INSTANCE | jq -r '.Instances[0].InstanceArn') \
  --target-type ACCOUNT \
  --target-id $(echo $AWS_SSO_CURRENT_IDENTITY | jq -r '.AccountId') \
  --principal-type USER \
  --principal-id $(echo $AWS_SSO_CURRENT_PRINCIPAL | jq -r '.Arn') \
  --permission-set-arn $(echo $AWS_SSO_PERMISSION_SET | jq -r '.PermissionSetArn') \
  --profile=sso-admin

  {{< /tab >}}
{{< /tabpane >}}

### Console Steps

To assign permissions to SSO-enabled users in AWS, use AWS IAM Identity Center (formerly AWS SSO). This approach is recommended for organizations using SSO, as it centralizes access management and leverages short-term credentials for improved security.

Start by [Creating a Permission Set](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtocreatepermissionset.html) - name this new Permission Set **ExampleCustomViewerPermissionSet**, and select the **ViewOnlyAccess* [Managed Policy](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetcustom.html#permissionsetsampconcept) to grant this Permission set with view only access.

Next, [Assigning Users](https://docs.aws.amazon.com/singlesignon/latest/userguide/user-assignments.html) to this Permission Set and AWS Account combination to allow users to sign-on assuming these permissions through SSO login. For more information, see [User Guide for AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) or

* [Create Account Assignment](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreateAccountAssignment.html)
* [Create Permission Sets](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreatePermissionSet.html)
* [Attach Managed Policies to Permission Sets](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_AttachManagedPolicyToPermissionSet.html)


## Legacy Account Setup
If we do not want to enable AWS Identity Center, and utilize SSO for CLI authentication, then we can also restrict AWS IAM Account access to developer machines only. These steps are also useful for creating Service Accounts required by AWS Services (not to be used by developers).

### CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang="Shell" >}}

export AWS_ACCOUNT=[INSERT AWS ACCOUNT ID]
export DEVELOPER_IP_CIDR=[INSERT IP CIDR]

**1. (Optional) Create an IP Restriction Policy**

cat > ip-restrict-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": "iam:*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": "${DEVELOPER_IP_CIDR}"
        }
      }
    }
  ]
}
EOF

Create the policy:

aws iam create-policy \
  --policy-name IPRestrictPolicy \
  --policy-document file://ip-restrict-policy.json

**2. Create the Service Principal (IAM User)**

aws iam create-user --user-name my-example-legacy-principal

**3. Attach the IP Restriction Policy (if created) and IAMFullAccess**

aws iam attach-user-policy \
  --user-name my-example-legacy-principal \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/IAMFullAccess

# If you created the custom IP restriction policy, attach it as well:
aws iam attach-user-policy \
  --user-name my-example-legacy-principal \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/IPRestrictPolicy

**4. Create Access Keys for CLI Use**

aws iam create-access-key --user-name my-example-legacy-principal

# Save the AccessKeyId and SecretAccessKey securely for CLI configuration.

{{< /tab >}}
{{< tab header="CLI (Destroy)" lang="Shell" >}}
**1. Detach Policies from the User**

aws iam detach-user-policy \
  --user-name my-example-legacy-principal \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/IAMFullAccess

aws iam detach-user-policy \
  --user-name my-example-legacy-principal \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/IPRestrictPolicy

**3. Delete the User**

aws iam delete-user --user-name my-example-legacy-principal

**4. (Optional) Delete the Custom Policy**

aws iam delete-policy \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/IPRestrictPolicy
{{< /tab >}}
{{< tab header="Terraform" lang="HCL" >}}
## ---------------------------------------------------------------------------------------------------------------------
## AWS PROVIDER
##
## Configures the AWS provider with CLI Configured Authentication.
## ---------------------------------------------------------------------------------------------------------------------
provider "aws" {
  alias   = "accountgen"
}

##---------------------------------------------------------------------------------------------------------------------
## AWS SERVICE ACCOUNT MODULE
##
## This module provisions an AWS service account along with associated roles and security groups.
##
## Parameters:
## - `service_account_name`: The display name of the new AWS Service Account.
## - `service_account_path`: The new AWS Service Account IAM Path.
## - `roles_list`: List of IAM roles to bing to new AWS Service Account.
##
## Providers:
## - `aws.accountgen`: Alias for the AWS provider for generating service accounts.
##---------------------------------------------------------------------------------------------------------------------
module "aws_service_account" {
  source  = "github.com/sim-parables/terraform-aws-service-account"

  service_account_name = "legacy-service-account"
  service_account_path = "example"

  providers = {
    aws.accountgen = aws.accountgen
  }
}
{{< /tab >}}
{{< /tabpane >}}

### Console Steps

#### Security Policy
Create a new Policy with [IP Restrictions](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_deny-ip.html) so only users in our security group can interact with AWS. This is helpful incase our security tokens are compromised, so even with unauthorized use malicuous actors still wont be able to access our AWS account on another IP address.

#### IAM Group
Create a [New IAM Group](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html) which we can bind the IP Whitelist policy above to.

#### IAM User
Create a [New IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) **Without** access to AWS Management Console - this user should only be able to access AWS through CLI.

* [Add User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console) to the the new IAM Group to bind the IP Whitelist policy

* Add the `IAMFullAccess` role to this user's [Permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console)

* Create a [CLI Access Credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey), and don't leave page until the Shared Credentials have been completed in the following steps