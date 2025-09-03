---
title: Github
date: 2024-10-17
description: >
  Setting up GitHub is essential for managing code version control, enabling seamless collaboration, and integrating with CI/CD pipelines for efficient development workflows.
weight: 10
categories: [Setup]
tags: [Github]
---

Having a GitHub organization is crucial for Data Practioners and Businesses because it centralizes and manages code repositories under one cohesive structure. It allows for better collaboration by giving multiple users access to shared projects while maintaining control over permissions and visibility. Instead of individual repositories scattered across personal accounts, a Github Organization allows projects to be consolidated in a secure and  professional environment. This structure is essential for large projects where multiple contributors, such as developers, designers, and project managers, need to collaborate efficiently while ensuring that intellectual property and source code are well-organized and protected.

Additionally, a GitHub organization simplifies permission management and enables role-based access control. You can assign specific roles to team members—like admin, maintainer, or contributor—ensuring that only authorized personnel can make changes to critical repositories or infrastructure. This is particularly important for scaling teams, where keeping track of individual contributions and access becomes more complex. The organization also integrates seamlessly with CI/CD pipelines, security tools, and other DevOps practices, ensuring consistent development practices across teams. Overall, a GitHub organization fosters better collaboration, security, and project management, which is critical for the success of modern development efforts.

# Getting Started

Creating a Github Organiziation can be done free of charge, and only requires that a Github account has already been created. If you're new to Github, then you can register [here](https://docs.github.com/en/get-started/start-your-journey/creating-an-account-on-github). When ready, follow these instructions for [Creating a New Organization from Scratch](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch).

## Install CLI

The **GitHub CLI (Command-Line Interface)** allows developers to interact with GitHub directly from the terminal, streamlining key workflows such as managing repositories, issues, and pull requests, as well as triggering CI/CD pipelines via GitHub Actions. With the CLI, you can authenticate, review code, merge pull requests, manage project settings, and gain insights into repositories—all without switching to the web interface.

Please visit the [Official Github CLI Installing Instructions](https://github.com/cli/cli?tab=readme-ov-file#installation) to review the install steps. Github (both the platform and the CLI) also require the standalone `git` cli to be installed. If you haven't already done so, please also follow [Github's git Install Instructions](https://github.com/git-guides/install-git) to install the  git CLI.

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="CLI" lang=Shell >}}
  # Login to Github Through CLI
  gh auth login

  # [Optional] rerfresh login needed sometimes
  gh auth refresh -h github.com -s user

  #https://docs.github.com/en/rest/users/users?apiVersion=2022-11-28#get-the-authenticated-user
  export GH_USER=$(gh api \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /user \
    --jq ".login")
  export GH_EMAIL=$(gh api \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    /user/emails \
    --jq ".[0].email")

  git config global.user $GH_USER
  git config global.email $GH_EMAIL
  {{< /tab >}}
{{< /tabpane >}}



## Repository

A source code repository to store your code base isn't a requirement to starting any new project right off the hop, but is a requirement when the project begins to scale and needs more team work involvement, or the project lifecycle is moving from POC into a MVP life stage. Creating and using Git repositories is likely common knowledge to many, but here's a quick refresher on creating a repository with Github.


{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="Web" >}}
  Go to Github's Adding locally hosted code to Github for more details on how to start.
  https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github#initializing-a-git-repository
  {{< /tab >}}
  {{< tab header="CLI" lang=Shell >}}
  # Create a new Python package scaffolding
  pip install --upgrade pyscaffold
  putup example_project

  # Add a README to begin describing what is the purpose of the project
  cd example_project
  cat <<EOT > README.md
    # Example Project
    This is a repository to store the code base for this example project
    ## Purpose
    Example learning materials
  EOT

  # Create the new github repository - all gh CLI to initialize local git
  # https://docs.github.com/en/github-cli/github-cli/quickstart#creating-a-repository
  gh auth refresh -h github.com -s user
  gh repo create

  {{< /tab >}}
{{< /tabpane >}}


## Environments

Github Environments are useful components within the Github Organization for isolating different working areas within CI/CD Deployment pipelines (or Github Action Workflows). Some common use case where environments become necessary is restricting Github Actions from being able to spin up costly cloud resources, or executing long running workflows which can deplete your Github Organization's free minutes. At OpenVERS, we recommend creating one Github Organization Environment to start

> **Production** This environment will be locked to only admins being able to sign-off on Github Action Workflows
> running. This is the protective layer in CI/CD where we will allow for infrastructure to be spun up - making
> sure that human interaction is required.

{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="Web" >}}
  Go to Github's Creating an Environment for more details on how to start.
  https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment#creating-an-environment
  {{< /tab >}}
  {{< tab header="CLI" lang=Shell >}}
  # Ensure current working directory is set in correct repo
  cd example_project

  # Create environment
  export GH_REPO=$(gh repo view --json name -q ".name")
  export GH_ORG=$(gh org list --limit 1) # Assumes only one organization owned by GH User
  gh api --method PUT \
    -H "Accept: application/vnd.github+json" \
    repos/$GH_ORG/$GH_REPO/environments/production

  {{< /tab >}}
{{< /tabpane >}}


### Secrets

GitHub Secrets are vital for securing sensitive information like API keys, access tokens, passwords, and other credentials used in automated workflows, such as continuous integration (CI) and continuous deployment (CD). By storing these values in encrypted environments within repositories or GitHub Actions, secrets help protect against accidental exposure of sensitive data in code or public logs. This is crucial for maintaining security, as it prevents unauthorized access to critical infrastructure or services. Additionally, GitHub Secrets enable teams to manage and update credentials securely without embedding them directly in codebases, ensuring that workflows remain safe, flexible, and compliant with best security practices.

> Sensitive credentials require high security scrutiny because their exposure can lead to unauthorized access,
> data breaches, and compromised systems, putting critical assets and confidential information at risk. Follow
> Github's [Security Hardening Best Practices](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) for more details. OpenVERS guides utilize OpenID Connect
> where possible as the "Best in Class" security hardening practice.


{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="Web" >}}
  Go to Github's Using Secrets in Github Actions for more details on how to configuring secrets.
  https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-an-organization
  {{< /tab >}}
  {{< tab header="CLI" lang=Shell >}}
  # Create organization secret
  # https://cli.github.com/manual/gh_secret_set
  gh auth refresh -h github.com -s user
  export GH_ORG=$(gh org list --limit 1) # Assumes only one organization owned by GH User

  # Paste secret value for the current repository in an interactive prompt
  gh secret set EXAMPLE_SECRET --org $GH_ORG --visibility all

  {{< /tab >}}
{{< /tabpane >}}


### Variables

GitHub Variables are crucial for creating dynamic and reusable workflows in GitHub Actions, allowing developers to manage configuration values that are frequently used across different workflows or environments. Unlike secrets, which are specifically designed to protect sensitive information, variables store non-sensitive data such as environment settings, deployment configurations, or feature flags. They improve workflow flexibility by centralizing and standardizing values that may change depending on the environment (e.g., development, staging, production), making workflows easier to maintain and scale. By using GitHub Variables, teams can avoid hardcoding values in workflows, reducing duplication and minimizing the risk of errors when updating configurations across multiple workflows. This improves efficiency, consistency, and maintainability in continuous integration and deployment pipelines.


{{< tabpane right=true >}}
  {{% tab header="**Examples**:" disabled=true /%}}
  {{< tab header="Web" >}}
  Go to Github's Using Secrets in Github Actions for more details on how to configuring secrets.
  https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-an-organization
  {{< /tab >}}
  {{< tab header="CLI" lang=Shell >}}
  # Create organization secret
  # https://cli.github.com/manual/gh_secret_set
  gh auth refresh -h github.com -s user
  export GH_ORG=$(gh org list --limit 1) # Assumes only one organization owned by GH User

  # Paste secret value for the current repository in an interactive prompt
  gh variable set EXAMPLE_SECRET --org $GH_ORG --visibility all

  {{< /tab >}}
{{< /tabpane >}}