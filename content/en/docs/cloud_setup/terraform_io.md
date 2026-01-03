---
title: Terraform.io
date: 2024-10-17
description: >
  Terraform.io platform service enables teams to collaborate on cloud agnostic infrastructure as code (IaC) workspaces.
weight: 11
categories: [Setup]
tags: [Terraform]
---

Terraform is a powerful infrastructure-as-code (IaC) tool that enables users to automate the provisioning, management, and scaling of infrastructure across various cloud providers such as AWS, Azure, and Google Cloud, as well as on-premise environments. With its declarative configuration language, Terraform allows users to define infrastructure in reusable code that can be versioned and shared, ensuring consistency and reducing manual errors. One of its key capabilities is the use of state files, which track the infrastructure's current state, enabling Terraform to apply only necessary changes when updating resources. Terraform’s provider-agnostic architecture supports a wide range of services, making it ideal for multi-cloud strategies and hybrid infrastructures. It also integrates seamlessly with CI/CD pipelines, enabling teams to automate infrastructure updates, enforce security policies, and collaborate effectively.

# Getting Started

Creating a Hashicorp Terraform account can be done free of charge, and only requies that a Github account has already been created. More details about registering with Github can be found in our [Github Setup Instructions](docs/cloud_setup/github). When ready, follow these instructions for [Create a New Hashicorp Terraform Account](https://app.terraform.io/public/signup/account?product_intent=terraform).


## Install CLI

The **Terraform CLI** is the primary tool for building infrastructure on cloud platforms. It allows users to define, plan, and apply infrastructure changes across various cloud providers and on-premise systems. The CLI also helps manage state files, configure variables, and track dependencies between resources, ensuring efficient, consistent, and automated infrastructure management.

Please visit the [Official Terraform CLI Installing Instructions](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform) to review the install steps. For the purposes of this example, please make sure Docker Engine in installed locally (follow install instructions at [Terraform/Docker Quick Start](https://developer.hashicorp.com/terraform/tutorials/docker-get-started/install-cli#quick-start-tutorial)).

## Infrastructure as Code (IaC)

Terraform configuration is a declarative way to define and manage infrastructure using human-readable configuration files, typically written in HashiCorp Configuration Language (HCL) or JSON. Once the configuration is written, Terraform can automatically provision and manage infrastructure based on these definitions, ensuring consistency across environments while reducing manual intervention and errors.

Here's a short example on connecting with Terraform.io to create a new workspace for our terraform infracture code, and writing a short terraform example with Docker to spin up an NGINX server. After the server has been spun up locally, we then destroy the resources as good practice for any example use cases.

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang=Shell >}}
  # Create new terraform.io workspace
  terraform login
  
  export TF_WORKSPACE=example-workspace
  export TF_TOKEN=$(cat \
    /home/server/.terraform.d/credentials.tfrc.json | \
    jq -r '.["credentials"]["app.terraform.io"]["token"]'
  )
  export TF_ORGANIZATION=$(curl \
    --header "Authorization: Bearer $TF_TOKEN" \
    --header "Content-Type: application/vnd.api+json" \
    --request GET \
    https://app.terraform.io/api/v2/organizations | \
    jq -r '.["data"][0]["id"]'
  ) # https://developer.hashicorp.com/terraform/cloud-docs/api-docs/organizations
  
  terraform workspace new $TF_WORKSPACE

  # Create example exercise path
  mkdir $TF_WORKSPACE
  cd $TF_WORKSPACE

  cat <<<EOT > maint.tf
    terraform {
      required_providers {
        docker = {
          source  = "kreuzwerker/docker"
          version = "~> 3.0.1"
        }
      }

      backend "remote" {
        # The name of your Terraform Cloud organization.
        organization = "$TF_ORGANIZATION"

        # The name of the Terraform Cloud workspace to store Terraform state files in.
        workspaces {
          name = "$TF_WORKSPACE"
        }
      }
    }

    provider "docker" {}

    resource "docker_image" "nginx" {
      name         = "nginx"
      keep_locally = false
    }

    resource "docker_container" "nginx" {
      image = docker_image.nginx.image_id
      name  = "tutorial"

      ports {
        internal = 80
        external = 8000
      }
    }
  EOT

  # Spin up example tf infrastructure
  terraform init
  terraform plan
  terraform apply --auto-approve

  # Spin down example tf infrastructure
  terraform destroy --auto-approve

  {{< /tab >}}
{{< /tabpane >}}

Proper Terraform configurations are modular, allowing developers to reuse code and organize resources logically. It can include variables for flexibility, resource blocks to define specific infrastructure components, and modules for grouping related resources. OpenVERS endorses [Don't Repeat Yourself (DRY)](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/) principals when building infrastructure, and follows the module pattern outlined by [Hashicorp Module Structure](https://developer.hashicorp.com/terraform/tutorials/modules/module-create#module-structure).

```

.
├──modules
│  ├──example_module_A
│  │  ├──resources.tf
│  │  ├──variables.tf
│  │  └──output.tf
│  └──example_module_A
│     ├──resources.tf
│     ├──variables.tf
│     └──output.tf
│
├──test
│  ├──main.tf
│  ├──variables.tf
│  ├──output.tf
│  └──terraform.tfvars
│
├──README.md
├──resources.tf
├──variables.tf
└──output.tf

```

## Orchestration

Terraform orchestration becomes even more powerful and streamlined, particularly for managing complex, multi-environment, or multi-account infrastructures. Terragrunt is a wrapper around Terraform that simplifies the orchestration of infrastructure by automating repetitive tasks and enforcing best practices. It helps manage and maintain consistent configurations across multiple Terraform modules, handling dependencies and remote state management more efficiently.

By using Terragrunt, teams can reduce boilerplate code, avoid duplication, and more easily orchestrate large infrastructure deployments across environments like development, staging, and production. It automatically handles resource dependencies, ensuring that Terraform runs in the right order and across different cloud environments or accounts. Terragrunt also allows for centralized configuration, reducing the manual effort of updating individual environments, making it easier to manage infrastructure changes at scale. This enhanced orchestration ensures smoother, more consistent infrastructure provisioning and management across various environments and teams.

Please visit the [Official Terragrunt CLI Installing Instructions](https://terragrunt.gruntwork.io/docs/getting-started/install/#install-via-a-package-manager) to review the install steps. Below is a continuation of the Docker terraform example above where we can use Terragrunt to orchestrate a simple single enviornment configuration deployment for our NGINX server.

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang=Shell >}}
  # Enter example exercise path
  cd $TF_WORKSPACE

  # Create a terragrunt configuration for our main.tf infrastructure
  cat <<<EOT > terragrunt.hcl
    terraform {
      source = "${path_relative_from_include()}/main"
    }
    
    inputs = {
      name = "terrafgrunt-example"
    }
  EOT

  # Spin up example with terragrunt
  terragrunt init
  terragrunt plan
  terragrunt apply --auto-approve

  # Spin down example with terragrunt
  terragrunt destroy --auto-approve

  {{< /tab >}}
{{< /tabpane >}}



