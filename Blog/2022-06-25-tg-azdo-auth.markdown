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

## Authorization

## Authentication

### HTTP Extraheader

### Inject PAT into Git Links using Regex

### Git Credential Manager Core

## Additional Links and references
