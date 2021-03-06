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

```terraform
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
```

What's happening here is the following:

1. A project is created in Azure DevOps
2. A group gets created within the newly created project to manage Read permissions on repository and project-level
3. All service users will be processed to find the current's project Build Service
4. This account is added to the defined project-level group

This will effectively allow the pipeline to access all repositories from the Library project.

## Authentication

When the pipeline is executed and the code is checked out from the remote repository the build agent must authenticate against the remote. This can be achieved using either OAuth methods or a Token in the context of Azure Repos. As stated, we're going to use the Access Token from the pipeline that we need to inject into all git operations somehow. To do so, I tested some approaches (I guess there are a couple more that I could not come up with).

### HTTP Extraheader

Reading the [Official Git Documentation](https://git-scm.com/docs/git-config#Documentation/git-config.txt-httpextraHeader), you'll find a method that allows git to add additional HTTP Headers into git operations. This seems to be the correct approach for our example and is also documented extensively within [Microsoft Docs](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows#use-a-pat).

Setting this up in a YAML pipeline is relatively easy as show below:

```YAML
- bash: |
          git config --global http.https://dev.azure.com/<organization>/Library/_git/composite-configuration.extraheader "AUTHORIZATION: bearer $(System.AccessToken)"
        displayName: 'Set Accesstoken in git extraheader'
```

Unfortunately, this solution does not scale well as the header needs to be modified for every git link throughout the configuration. Therefore, it was neglected for the given task.

### Inject PAT into Git Links using Regex

Going foward, I stumbled upon an idea to just do some regex-magic on the git links within the Terragrunt.hcl file(s) to replace all links with another one containing the Access token as shown in the snippet below.

```YAML
- bash: |
    find $(Build.SourcesDirectory)/ -type f \( -name 'terragrunt.hcl'\) -exec sed -i 's~git::https://dev.azure.com~git::https://$(System.AccessToken)@dev.azure.com~g' {} \;
  displayName: "Inject System Access Token into Git links"
```

This works great as long as there are no further git links within the Terraform configuration downloaded. As this is not the case for our given case, this method was neglected as well.

### Git Credential Manager Core

[GCM Core](https://github.com/GitCredentialManager/git-credential-manager) is a Git Credential Manager implemented in .net Core that allows for unified Credential Management for git for all computing platforms (Windows, Linux, macOS).

For the given task, GCM has been added to the Built Agents Docker image by adding the following snippet into the Dockerfile.

```Dockerfile
# Install Git Credential Manager Core
RUN curl -LO https://raw.githubusercontent.com/GitCredentialManager/git-credential-manager/main/src/linux/Packaging.Linux/install-from-source.sh && \
    echo y | sh ./install-from-source.sh && \
    git-credential-manager-core configure
```

Even though, GCM Core supports various secure mechanisms to store credentials, it was decided to work with plaintext credentials as those are only stored as long as the pipeline runs where the token is readable anyways.

To store the credentials beforehand, a [PowerShell script](https://gist.github.com/techdecline/24ebc99a0fa4d25b6c8527ac8414412b) has been created that composes a working Git Credential file.

```Powershell
param (
    [String]$OrganizationName = $($Env:SYSTEM_COLLECTIONURI -replace ".$") ,
    [String]$UserName = "serviceuser@organization.com", # this is just a dummy value
    [String]$PAT = $Env:SYSTEM_ACCESSTOKEN,
    [String]$StorePath = $PWD
)

# git config
. git config --global credential.interactive false
. git config --global credential.credentialStore plaintext
. git config --global credential.plaintextStorePath $StorePath
. git config --global credential.azreposCredentialType pat

$sb = [System.Text.StringBuilder]::new()
[void]$sb.AppendLine( $PAT )
[void]$sb.AppendLine( "service=$OrganizationName" )
[void]$sb.AppendLine( "account=$UserName" )

$gcmPlaintextPath = Join-Path -Path $StorePath -ChildPath "git/https/dev.azure.com/$(($OrganizationName -split "/")[-1])/$UserName.credential"
$null = New-Item $gcmPlaintextPath -Force
$null = Set-Content -Path $gcmPlaintextPath -Value $sb.ToString()
```

Within the pipeline the script gets called with predefined variables as parameters.

```YAML
- powershell: |
    . ./scripts/New-GcmPlainTextGitConfig.ps1
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
    SYSTEM_COLLECTIONURI: $(System.CollectionUri)
  displayName: "Setup Git Credential Manager Core with Plaintext Credentials for Azure DevOps"
```

This way all git links derived from the Azure DevOps organization will use the pipeline's access token without additional configuration. Additionally, the solution could theoretically be extended to store additional credentials for other organizations and utilize [encrypted methods](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/credstores.md) for storing credentials.

## Verdict

In case your configuration is stored in single repository, using the HTTP extraheader is the go-to option thereas using the regex-method yields best results when your configuration is stored in repositories that do not contain additional remote calls.

Using Git Credential Manager Core gives you the highest amount of flexibility for the cost of updating your deployment image beforehand.
