---
title: "Trying to setup an Azure DevOps organization using Terraform"
date: 2021-03-10T11:10:02+01:00
tags: ["azure", "devops", "terraform", "aad"]
draft: true
---

> **Give me the code**   
> If you don't care about my writings, I have upload the source code on my Github:    
> https://github.com/karlospn/provisioning-an-azdo-org-with-terraform


These past few days I have been tinkering with the Azure DevOps provider for Terraform and I was pleasant surprised with it.    
So on **today's post I want to try to use it for automating the setup of an Azure DevOps organization**.   


> You might also be interested in these post I wrote a few months ago about automating the app registration process using the Terraform Azure Active Directory provider: 
> https://www.mytechramblings.com/posts/automate-azure-ad-app-registration-using-terraform/


# Prerequisites

There are some prerequisites I have done before start building this scenario to speed things up. The prerequisites are the following ones:

- I have created an AzDo organization.
- I have an existing Azure Active Directory, and I have populated it with some users and groups.
  - This prerequisite is key because we're not going to create new users on my AzDo organization. We're going to use the existing AAD groups and enrolls the users from those groups into our AzDo organization.
  - Using the users from AAD it's a more realistic scenario than creating some new users directly on AzDo.
- I have connected my Azure DevOps organization into the AAD directory.

![azdo-connect-aad](/img/azdo-connect-aad.png)

  

# Scenario to build

I'm going try to build a more or less "realistic" scenario.    
So we're going to build the following scenario:

- We have 2 development teams (the commercial dev team and the sales dev team).
- We have 1 managers team.
- Those 3 teams have been created as groups on my Azure Active Directory.
- We want to enroll them on my AzDo organization so they can start working as soon as possible.
  

1 - We need to enroll those 3 AAD groups into my AzDo organization and assign a license to every member of the group.   
2 - We need to create a Team Project for each development team. The managers team doesn't need a Team Project.   
3 - We need to add permissions to each AAD group so each team can start working on his respective Team Project.   
4 - We need to create the git repositories for all the applications that each team are going to develop. Also for every git repository we create, we want to set some branch policies in the main branch.   

# Step 0 - Setup the Terraform providers


## 1. Azure DevOps Terraform provider

The Azure DevOps provider only supports a PAT (personal access token) for authenticating to Azure DevOps.   

>More info about it here: https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs/guides/authenticating_using_the_personal_access_token

So I guess I'm going to create PAT on my AzDo org.

> The PAT has to be created it manually via the AzDo UI.    
> Right now there is no other way to do it, but be aware  that a PAT lifecicle management API is going to be available in a near future: https://docs.microsoft.com/en-us/azure/devops/release-notes/2021/sprint-183-update#pat-lifecycle-management-api-private-preview


The provider configuration will look like this:
```yaml
provider "azuredevops"{
    org_service_url         = "https://dev.azure.com/cpn"
    personal_access_token   = "the_pat_goes_here"
}
```

**You don't want to hardcode the PAT on the Terraform file.**    
Use variables or variables files or Terragrunt or whatever you want, but do not hardcode the PAT creds in here.   

## 2. Azure Active Directory Terraform provider

I'm going to need to use the Azure AD Terraform provider, but why? 
- The AzDo provider cannot access the AAD to retrieve the AAD groups.

So I'm going to use this provider to retrieve the AAD groups info and use the output with the AzDo provider.   

To authenticate against my AAD I'm going to register a new App on my AAD with some Graph permissions and create a Service Principal with a client secret .   
> If you want to read more about what you need, you can go here: https://www.terraform.io/docs/providers/azuread/guides/service_principal_client_secret.html


The provider configuration will look like this:
```yaml
provider "azuread" {
    client_id         = "ba4d0620-0522-4ada-b0b6-0cdd8cfaeae7"
    client_secret     = "my_secret_goes_here"
    tenant_id         = "my_tenant_goes_here"
}
```

**You don't want to hardcode the client secret on the Terraform file.**    
Use variables or variables files or Terragrunt or whatever you want, but do not hardcode the PAT creds in here.


# Step 1 - Enroll the AAD groups into my AzDo org

We have 3 groups on my Azure Active Directory Team.
  - it-commercial-team: These group contains the developers for the commercial development team.
  - it-sales-team: These group contains the developers for the sales team.
  - it-managers: These group contains the bosses from both teams.

And each group contains some AAD users.

That's the look of the 3 teams on my AAD this:

<ADD FOTO HERE>

And as you can see every team as some user. For example if we take a look at the it-commercial-team, we can see:

<ADD FOTO HERE>

 We want to enroll those 3 AAD groups into my AzDo organization and assign all the users from those groups a license. 
 - I'm going to assign a "Basic" license to the commercial and the sales team.
 - I'm going to assign a "Stakeholder" license to the managers team.


Instead of creating a single Terraform file with the enrollment of these 3 groups, I'm going to built a Terraform module so I can reuse it to enroll each team. 

The end result looks like this.

- That's the main.tf file.
- The main.tf is using the "add-entitlement-to-group-users" module to enroll the users.

```yaml
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }

    azuread = {
      source = "azuread"
      version = ">=1.4.0"
    }
  }
}

provider "azuredevops"{
    org_service_url = var.org_service_url
    personal_access_token = var.personal_access_token
}

provider "azuread" {
    client_id = var.aad_client_id
    client_secret = var.aad_client_secret
    tenant_id     = var.aad_tenant_id
}

## Add entitlements to all users from the AAD it-sales-team
module "add-entitlement-to-sales-team-group-users" {
    source      = "../modules/add-entitlement-to-group-users"
    aad_users_groups = ["it-sales-team"]
    license_type = "basic"
}

## Add entitlements to all users from the AAD it-commercial-team
module "add-entitlement-to-commercial-team-group-users" {
    source      = "../modules/add-entitlement-to-group-users"
    aad_users_groups = ["it-commercial-team"]
    license_type = "basic"
}

## Add entitlements to all users from the AAD it-managers-team
module "add-entitlement-to-managers-team-group-users" {
    source      = "../modules/add-entitlement-to-group-users"
    aad_users_groups = ["it-managers-team"]
    license_type = "stakeholder"
}

```
- And those are the module files(variables.tf, output.tf and main.tf)

_For ease the reading, I have compacted all the module files on a single markdown block code_

```yaml
#/modules//add-entitlement-to-group-users/variables.tf

variable "license_type" {
    type = string
}

variable "aad_users_groups" {
    type = list(string)
}


#/modules/add-entitlement-to-group-users/main.tf
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }
    
    azuread = {
      source = "azuread"
      version = ">=1.4.0"
    }
  }
}

## Flattening the members from the different groups
locals {
  members = distinct(flatten([
    for s in data.azuread_group.aad_group: [
      for m in s.members:{
        item = m
      }
    ]
  ]))
}

## Get group info from azure ad
data "azuread_group" "aad_group" {
  for_each  = toset(var.aad_users_groups)
  display_name     = each.value
}

## Get aad users from group
data "azuread_user" "aad_users" {
  for_each = { for x in local.members: x.item => x }
  object_id = each.value.item
}

## Add entitlement to users in the aad group
resource "azuredevops_user_entitlement" "user" {
  for_each = data.azuread_user.aad_users
  principal_name = each.value.user_principal_name
  account_license_type = var.license_type
}

#/modules/add-entitlement-to-group-users/output.tf
output "aad_users" {
    value = {
        for user in data.azuread_user.aad_users: 
            user.id => user.display_name
    }
}

```

Let me explain a little bit what we're doing on the module:

- First with the "azuread_group" data resource we retrieve the groups from the AAD.
- After that with the "azuread_user" data resource we  retrieve all the users from each AAD group.
- Finally with the "azuredevops_user_entitlement" we enroll every AAD user into our AzDo org.

Maybe someone is asking himself: "Why are you not using the Azure DevOps "Group Rules" feature?"    
It's true that the AzDo "Group Rules" ((https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/assign-access-levels-by-group-membership) allows us to do exactly the same thing we're doing in an easier way, but the answer is pretty simple the provider doesn't support it.

# Step 2 - Create a Team project for each team

- We want to create 2 Team Projects. 
  - One team project for the commercial development Team.
  - One team project for the sales development Team.

There is no need for a module here because it's pretty straightforward.

The terraform file, will look like this:

```yaml
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }
  }
}

provider "azuredevops"{
    org_service_url = var.org_service_url
    personal_access_token = var.personal_access_token
}

## Create team project for the commercial team
resource "azuredevops_project" "project-comm" {
  name       = "Commercial Team Project"
  description        = "Repository used by the Commercial IT Team"
  visibility         = "private"
  version_control    = "Git"
  work_item_template = "Agile"

  features = {
      "boards" = "disabled"
      "repositories" = "enabled"
      "pipelines" = "enabled"
      "artifacts" = "enabled"
  }
}

## Create team project for the sales team
resource "azuredevops_project" "project-sales" {
  name       = "Sales Team Project"
  description        = "Repository used by the Sales IT Team"
  visibility         = "private"
  version_control    = "Git"
  work_item_template = "Agile"

  features = {
      "boards" = "disabled"
      "repositories" = "enabled"
      "pipelines" = "enabled"
      "artifacts" = "enabled"
  }
}
```

Nothing remarkable to explain here, it's pretty self-explanatory.

# Step 3 - Add permissions

Let's make a quick recap:
- On step 1 we have enrolled those 3 AAD groups: it-commercial-team,it-sales-team, it-managers into our AzDo org.
- On step 2 we have created a team project for the 'it commercial team' and anothere team project for the 'it sales team'.   

Now in step 3 it's time to set the permissions in each team project. We're going to add these permissions:

- In the 'Commercial Team Project' we're going to set the following permissions:
  - The 'it-commercial-team' AAD group is going to be added into the 'Contributors' security group.
  - The 'it-managers' AAD group is going to be added into the 'Readers' security group

- In the 'Sales Team Project' we're going to set the following permissions:
  - The 'it-sales-team' AAD group is going to be added into the 'Contributors' security group.
  - The 'it-managers' AAD group is going to be added into the 'Readers' security group


I'm going to built another Terraform module so I can reuse it to add permissions for each group.

The end result looks like this.

- That's the main.tf file.
- The main.tf is using the "add-aad-groups-to-azdo-team-project-sec-group" module to add the permissions for every group in every team project.

```yaml
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }

    azuread = {
      source = "azuread"
      version = ">=1.4.0"
    }
  }
}

provider "azuredevops"{
    org_service_url = var.org_service_url
    personal_access_token = var.personal_access_token
}

provider "azuread" {
    client_id = var.aad_client_id
    client_secret = var.aad_client_secret
    tenant_id     = var.aad_tenant_id
}


## Add the commercial teams AAD group as contributor on the commercial team project
module "add-comm-group-to-azdo-sec-group" {
    source      = "../modules/add-aad-groups-to-azdo-team-project-sec-group"
    project_name =  "Commercial Team Project"
    azdo_group_name = "Contributors"
    aad_users_groups = ["it-commercial-team"]
}

## Add the sales teams AAD group as contributor on the sales team project
module "add-sales-group-to-azdo-sec-group" {
    source      = "../modules/add-aad-groups-to-azdo-team-project-sec-group"
    project_name = "Sales Team Project"
    azdo_group_name = "Contributors"
    aad_users_groups = ["it-sales-team"]
}


## Add the managers AAD group group as readers on the commercial team project
module "add-manager-group-to-comm-azdo-sec-group" {
    source      = "../modules/add-aad-groups-to-azdo-team-project-sec-group"
    project_name =  "Commercial Team Project"
    azdo_group_name = "Readers"
    aad_users_groups = ["it-managers-team"]
}


## Add the managers AAD group as readers on the sales team project
module "add-manager-group-to-sales-azdo-sec-group" {
    source      = "../modules/add-aad-groups-to-azdo-team-project-sec-group"
    project_name = "Sales Team Project"
    azdo_group_name = "Readers"
    aad_users_groups = ["it-managers-team"]
}

```
- And the module files:

_For ease the reading, I have compacted all the module files on a single markdown block code_

```yaml
#/modules/add-aad-groups-to-azdo-team-project-sec-group/variables.tf
variable "project_name" {
    type = string
}

variable "azdo_group_name" {
    type = string
}

variable "aad_users_groups" {
    type = list(string)
    default = []
}


#/modules/add-aad-groups-to-azdo-team-project-sec-group/main.tf
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }
    
    azuread = {
      source = "azuread"
      version = ">=1.4.0"
    }
  }
}

## Get project info
data "azuredevops_project" "project" {
  name = var.project_name
}

## Get group info from azure ad
data "azuread_group" "aad_group" {
  for_each  = toset(var.aad_users_groups)
  display_name     = each.value
}

## Get group info from AzDo
data "azuredevops_group" "azdo_group" {
  project_id = data.azuredevops_project.project.id
  name       = var.azdo_group_name
}

## Link the aad group to an azdo group
resource "azuredevops_group" "azdo_group_linked_to_aad" {
  for_each  = toset(var.aad_users_groups)
  origin_id = data.azuread_group.aad_group[each.key].object_id
}

## Add membership
resource "azuredevops_group_membership" "membership" {
  group = data.azuredevops_group.azdo_group.descriptor
  members = flatten(values(azuredevops_group.azdo_group_linked_to_aad)[*].descriptor)
}
```

Let's also explain a little bit what we're doing in the module:

- With the "azuread_group" data resource we retrieve the groups from the AAD.
- With the "azuredevops_group" data resource we retrieve the security group.
- With the "azuredevops_group" we are linking the aad group to the azdo group.
- Finally with the "azuredevops_group_membership" we're adding the AAD group into the security group.

# Step 4 - Create git repositories and master branch policies

In the step 4 we're going to create the git repositories for all the applications that each team are going to develop.   
Also for every git repository we create, we want to set some branch policies in the main branch.   


We're going to create the following git repositories:

- In the Commercial Team Project we're going to create:
  - A git repository for a webapi + some branch policies on the main branch
  - A git repository for a SPA + some branch policies on the main branch

- In the 'Sales Team Project we're going to create:
  - A git repository for a WebAPI + some branch policies on the main branch

I'm going to built another Terraform module so I can reuse it when creating the different git repositories.   

The end result looks like this.

- That's the main.tf file.
- The main.tf is using the "create-repository-and-branch-policies" Terraform module to add create the git repositories and the branch policies.

```yaml
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }
  }
}

provider "azuredevops"{
    org_service_url = var.org_service_url
    personal_access_token = var.personal_access_token
}

provider "azuread" {
    client_id = var.aad_client_id
    client_secret = var.aad_client_secret
    tenant_id     = var.aad_tenant_id
}

locals {
  comm_prj_name = "Commercial Team Project"
  sales_prj_name = "Sales Team Project"
}

## Create repository for commercial team apps
## Create repository for commercial team api
module "create-repository-and-policies-for-commercial-team-api" {
    source      = "../modules/create-repository-and-branch-policies"
    project_name = local.comm_prj_name
    repository_name = "comm-api"
}

## Create repository for commercial team ui
module "create-repository-and-policies-for-commercial-team-ui" {
    source      = "../modules/create-repository-and-branch-policies"
    project_name = local.comm_prj_name
    repository_name = "comm-ui"
}

## Create repository for sales team apps
## Create repository for sales team ui
module "create-repository-and-policies-for-sales-team-api" {
    source      = "../modules/create-repository-and-branch-policies"
    project_name = local.sales_prj_name
    repository_name = "sales-api"
}
```
- And the module source code:
  
_For ease the reading, I have compacted all the module files on a single markdown block code_
```yaml
#/modules/create-repository-and-branch-policies/variables.tf
variable "project_name" {
    type = string
}

variable "repository_name" {
    type = string
}


#/modules/create-repository-and-branch-policies/main.tf
terraform {
  required_providers {
    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.1.0"
    }
  }
}


## Get project info
data "azuredevops_project" "project" {
  name = var.project_name
}


## Create Git Repository
resource "azuredevops_git_repository" "repository" {
  project_id = data.azuredevops_project.project.id
  name       =  var.repository_name
  initialization {
    init_type = "Clean"
  }
}


## Start Branch permission block
resource "azuredevops_branch_policy_min_reviewers" "policy-min-reviewers" {
  project_id = data.azuredevops_project.project.id

  enabled  = true
  blocking = true

  settings {
    reviewer_count     = 1
    submitter_can_vote = false
    last_pusher_cannot_approve = true
    allow_completion_with_rejects_or_waits = false
    on_push_reset_approved_votes = true

    scope {
      repository_id  = azuredevops_git_repository.repository.id               
      repository_ref = azuredevops_git_repository.repository.default_branch
      match_type     = "Exact"
    }
  }
}

resource "azuredevops_branch_policy_comment_resolution" "policy-comment-resolution" {
  project_id = data.azuredevops_project.project.id

  enabled  = true
  blocking = true

  settings {

     scope {
      repository_id  = azuredevops_git_repository.repository.id               
      repository_ref = azuredevops_git_repository.repository.default_branch
      match_type     = "Exact"
    }
  }
}

## End Branch permission block

#/modules/create-repository-and-branch-policies/outputs.tf
output "repository_id" {
  value = azuredevops_git_repository.repository.id
}
```
The module needs no explanation, it's really simple.


# Next Steps

I could keep adding more and more steps into the scenario, for example I could built another module that creates the service connections to AWS or Azure so we are ready to deploy the apps whenever we want. 

But I think that this post it's a little bit longer than usual so I'm going to stop here.

In the end I'm pretty pleased with the provider. Some resources are still missing but overall it's pretty solid, to be honest it looks better that the Azure AD provider.

Also the provider offers some resources that I haven't tested yet, if after reading this post you're interested in trying it out for yourself, go take a look:
https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs



