---
title: OpenID Connect
date: 2024-11-03
description: >
  AWS - Modernized Security for GitOps & CI/CD where Passwords and Credentials are a Thing of the Past.
weight: 3
categories: [Setup, OpenID Connect]
tags: [AWS]
---

# OpenID Connect Configuration with Github

OpenID Connect (OIDC) is an authentication protocol that enables AWS to securely authenticate users and applications without requiring them to store and manage credentials, allowing for seamless integration with identity providers such as Google, Microsoft, Amazon and Github to providing a seamless and highly secure way to manage access to AWS resources.

## CLI Steps

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI (Create)" lang="Shell" >}}
mkdir ~/development/test/example_aws_oidc
cd ~/development/test/example_aws_oidc
  
# Setting
export GH_REPO="your-github-repo-name"

# Login to AWS root account
aws sso login --profile=sso-admin

# Create New User
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-user.html
export AWS_IAM_USER_ARN=$(aws iam create-user \
    --user-name $GH_REPO \
    --output text \
    --query 'User.Arn' \
    --profile=sso-admin)

# Create IAM Identity Provider for Github Actions
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-open-id-connect-provider.html
export AWS_IAM_IDENTITY_PROVIDER_ARN=$(aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "1b511abead59c6ce207077c0bf0e0043b1382612" > create-open-id-connect-provider.json \
  --query 'OpenIDConnectProviderArn' \
  --output text \
  --profile=sso-admin)

# Create IAM Policy for OIDC Access
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
cat <<EOT > aws_oid_sandbox_policy_document.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:GenerateCredentialReport",
        "iam:GenerateServiceLastAccessedDetails",
        "iam:Get*",
        "iam:List*",
        "iam:SimulateCustomPolicy",
        "iam:SimulatePrincipalPolicy"
      ],
      "Resource": "*"
    }
  ]
}
EOT
  
# Create IAM Trust Policy for IAM Role
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
cat <<EOT > aws_oidc_sandbox_role_trust_policy_document.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "$AWS_IAM_IDENTITY_PROVIDER_ARN"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:$GH_REPO/*:ref:refs/heads/main"
        },
        "ForAllValues:StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:iss": "https://token.actions.githubusercontent.com"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "$AWS_IAM_USER_ARN"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {}
    }
  ]
}
EOT

# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html
export AWS_OIDC_IAM_POLICY_ARN=$(aws iam create-policy \
  --policy-name example-oidc-sandbox-policy \
  --policy-document file://aws_oidc_sandbox_policy_document.json \
  --query 'Policy.Arn' \
  --output text \
  --profile=sso-admin)
  
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-role.html
export AWS_OIDC_IAM_ROLE_ARN=$(aws iam create-role \
  --role-name example-oidc-sandbox-role \
  --assume-role-policy-document file://aws_oidc_sandbox_role_trust_policy_document.json \
  --query 'Role.Arn' \
  --output text \
  --profile=sso-admin)
  
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/attach-role-policy.html
aws iam attach-role-policy \
  --policy-arn $AWS_OIDC_IAM_POLICY_ARN \
  --role-name example-oidc-sandbox-role \
  --profile=sso-admin
  {{< /tab >}}
  {{< tab header="CLI (Destroy)" lang="Shell" >}}
# Remove IAM Roles & Policies
aws iam detach-role-policy \
  --policy-arn $AWS_OIDC_IAM_POLICY_ARN \
  --role-name example-oidc-sandbox-role \
  --profile=sso-admin

aws iam delete-role \
  --role-name example-oidc-sandbox-role \
  --profile=sso-admin

aws iam delete-policy \
  --policy-arn $AWS_OIDC_IAM_POLICY_ARN \
  --profile=sso-admin

aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn \
  --profile=sso-admin
  {{< /tab >}}
{{< /tabpane >}}

## Console Steps

### Create Github OIDC IAM Account

The **Assume Role** Trust Policies will need to be bound to a specific IAM Account in order to have Github's CICD Action Workflows to both authenticate and assume roles to with enough permissions as required. We will create unique accounts for Read and Write role sets (similar to Terraform's Plan and Apply operations).

Create a [New IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) **Without** access to AWS Management Console - this user should only be able to access AWS through CLI. We'll repeat this step twice for the following accounts
  * github-cicd-service-principal-read
  * github-cicd-service-principal-readwrite

### Add Identity Provider

AWS prodives the following [Identity Provider Setup Instructions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html#manage-oidc-provider-console) on how to access and setup Identity Providers in AWS Console. However, Github's [Adding the Identity Provider to AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html#manage-oidc-provider-console) guide provides specific details for configuring OIDC federated access from Github's CICD Action Workflows to AWS.

Add a new [Identity Provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html#manage-oidc-provider-console) using the web console. Go to IAM > Identity Provider.
* Select OpenID Connect.
* Input https://token.actions.githubusercontent.com in the **Provider URL**
* Input sts.amazonaws.com in the **Audience** field

### Configure IAM Role and Trust Policies

Create new IAM roles to allow for Federated OIDC authentication with AWS. For additional details, see Github's [Configuring the Role and Trust Policy](github-cicd-service-principal-read).

Add a new [IAM Role](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/create_role.html) using the web console. Go to IAM > Roles.
For the *github-cicd-service-principal-**read*** account, name the associate role **github-cicd-service-principal-read-sts-role**.
* Attach the **IAMReadOnlyAccess** permission to the new role.
* Add the following [Trust Policy](https://aws.amazon.com/blogs/security/how-to-use-trust-policies-with-iam-roles/) to the new role.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "[COPY AWS IAM IDENTITY PROVIDER ARN]"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:[GITHUB ORG NAME]/*:ref:refs/heads/main"
                },
                "ForAllValues:StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:iss": "https://token.actions.githubusercontent.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "[COPY AWS IAM ACCOUNT FOR github-cicd-service-principal-read ARN]"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ],
            "Condition": {}
        }
    ]
}
```

For the *github-cicd-service-principal-**readwrite*** account, name the associate role **github-cicd-service-principal-readwrite-sts-role**.
* Attach the **IAMFullAccess** permission to the new role.
* Add the following [Trust Policy](https://aws.amazon.com/blogs/security/how-to-use-trust-policies-with-iam-roles/) to the new role.

```

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "[COPY AWS IAM IDENTITY PROVIDER ARN]"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:[GITHUB ORG NAME]/*:environment:production"
                },
                "ForAllValues:StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:iss": "https://token.actions.githubusercontent.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "[COPY AWS IAM ACCOUNT FOR github-cicd-service-principal-readwrite ARN]"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ],
            "Condition": {}
        }
    ]
}

```