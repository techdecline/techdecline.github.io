---
layout: post
title:  "Authenticating multiple Azure Repos in Terragrunt Pipelines on Azure DevOps (using Git Credential Manager Core)"
date:   2022-06-25 19:06:08 +0000
parent: Blog
categories: iac, automation, terraform
---
In a recent assignment, I had the challenge of authenticating multiple repositories from an Azure DevOps pipeline using just the pipelines System Access Token. This post demonstrates the steps and false assumptions that have lead to the final solution.

## Problem Statement

When working with Terraform or Terragrunt, you will probably run into scenarios where your code is stored in multiple private repositories as demonstrated within the chart.
![Multi-Repository Configuration](/images/multi-repo-terragrunt.png)

Given the fact that most of those repositories are marked as "private", downloading those requires authorization and authentication. By default, only the repository containing the pipeline file gets checked out whereas every other git operation needs to be configured correctly. Otherwise, downloading the modules at the Terraform Initialization phase fails as shown below.

``2022-05-31T06:03:53.3285001Z * error downloading 'https://dev.azure.com/<organization>/Library/_git/es-connectivity': /usr/bin/git exited with 128: Cloning into '/azp/_work/2/s/composite-configuration/.terragrunt-cache/6YQTAgdBopQSz0ArtaBEvpCE2IE/Bj6MNnx9ROQxV8ZhouXXEIcyaag'...
2022-05-31T06:03:53.3286636Z fatal: could not read Username for 'https://dev.azure.com': terminal prompts disabled``

## Explaination

As described within [Microsoft Docs](https://docs.microsoft.com/en-us/azure/devops/repos/git/auth-overview) you will need to configure authentication and authorization in a non-interactive manner to access git repositories outside the pipeline's scope.

## Authorization

Authorization means to allow a given entity to access certain resources. In the given case, we're going to use the pipeline's Personal Access Token (called [System.AccessToken](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#systemaccesstoken)) to download all required modules.

To do this, we must allow the exact Build Service Account to read project information and git repositories on the Library project. Below, you will find a working Terraform Snippet to set that up.

``
locals {
  azdo_org_name = regex("<https://dev.azure.com/(.*)/>$", data.azuredevops_client_config.cfg_azdo.organization_url)[0]
}

data "azuredevops_client_config" "cfg_azdo" {}

resource "azuredevops_project" "pr_azdo_infra" {
  name               = "InfrastructureLibrary"
  visibility         = "private"
  version_control    = "Git"
  work_item_template = "Agile"
  description        = "Project to store re-usable infrastructure components like Terraform modules, Ansible Playbooks, PowerShell Modules and such."
  features = {
    "testplans" = "disabled"
    # "artifacts" = "disabled"
    "boards"    = "disabled"
    "pipelines" = "disabled"
  }
}

data "azuredevops_project" "pr_azdo_platform" {
  project_id = var.azdo_project_id `
}

resource "azuredevops_group" "grp_infra_azdo_func_read" {
  scope        = azuredevops_project.pr_azdo_infra.id
  display_name = "_functional_Code_Read"
  description  = "Code Write permission for project. Created by Terraform"
}

resource "azuredevops_project_permissions" "perm_azdo_infra_read" {
  project_id = azuredevops_project.pr_azdo_infra.id
  principal  = azuredevops_group.grp_infra_azdo_func_read.id
  permissions = {
    GENERIC_READ = "Allow"
  }
}

resource "azuredevops_git_permissions" "git_azdo_infra_read" {
  project_id = azuredevops_project.pr_azdo_infra.id
  principal  = azuredevops_group.grp_infra_azdo_func_read.id
  permissions = {
    GenericRead           = "Allow"
    PullRequestContribute = "Allow"
  }
}

data "azuredevops_users" "user_azdo_svc" {
  origin        = "vsts"
  subject_types = ["svc"]
}

data "azuredevops_group" "grp_azdo_infra_read" {
  project_id = azuredevops_project.pr_azdo_infra.id
  name       = "Readers"
}

resource "azuredevops_group_membership" "gm_azdo_infra_read" {
  for_each = {
    for user in data.azuredevops_users.user_azdo_svc.users : user.display_name => user
    if user.display_name == "${data.azuredevops_project.pr_azdo_platform.name} Build Service (${local.azdo_org_name})"
  }
  group = azuredevops_group.grp_infra_azdo_func_read.descriptor
  members = [
    each.value.descriptor
  ]
}
``

What's happening here is the following:

1. A project is created in Azure DevOps
2. A group gets created within the newly created project to manage Read permissions on repository and project-level
3. All service users will be processed to find the current's project Build Service
4. This account is added to the defined project-level group

This will effectively allow the pipeline to access all repositories from the Library project.

## Authentication

### HTTP Extraheader

### Inject PAT into Git Links using Regex

### Git Credential Manager Core

## Additional Links and references
