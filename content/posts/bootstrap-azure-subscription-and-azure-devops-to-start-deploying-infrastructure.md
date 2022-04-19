---
title: "How to bootstrap Terraform and Azure DevOps to start deploying your infrastructure as code to Azure"
date: 2022-04-15T14:22:10+02:00
tags: ["azure", "devops", "terraform", "iac"]
description: "Deploying insfrastructure as code on Azure using Azure Pipelines and Terraform requires a minimal bootstrap process. This process can be done manually but you'll have to do it every time you want to start deploying resources into a new subscription. So, having some kind of automation seems the way to go here. And that's exactly what I want to show in this post, how to programmatically bootstrap an Azure subscription and an Azure DevOps project to start deploying Infrastructure as Code with Terraform."
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/bootstrapping-azure-subscription-and-azdo-project-for-terraform).

Nowadays almost every developer knows what infrastructure as code is.

In a nutshell, infrastructure as code (IaC) is the process of managing and provisioning computer resources through machine-readable definition files, instead of through physical hardware configuration or interactive configuration tools.

Deploying insfrastructure as code on Azure using Azure Pipelines and Terraform requires a minimal bootstrap process. The required steps are the following ones:
-  Create a Resource Group.
-  Create an Storage Account to store the Terraform state.
-  Create a Service Principal on our Azure Active Directory.
-  Assign a role to the SP with enough permissions to create new resources on Azure.
-  Store the SP credentials somewhere safe.
-  When an IaC pipeline is executed on Azure Pipelines, it needs to retrieve the SP credentials and create/update/delete the desired infrastructure on Azure using Terraform.

As you can see the required steps to start using IaC with Terraform and Azure Pipelines are quite simple and you could perfectly do it manually using the Azure Portal, but you'll have to do them every time you want to start deploying resources into a new subscription. So, having some kind of automation seems the way to go here.

And that's what I want to show in this post, how to programmatically bootstrap an Azure subscription and an Azure DevOps project to start deploying Infrastructure as Code with Terraform.

# Azure Bicep vs ARM Templates vs Terraform

When working with Azure we have 2 native options for IaC: Azure Resource Manager (ARM) templates or Azure Bicep.

ARM templates are essentially just JSON files with a few extended capabilities, describing the expected infrastructure to end up with, after applying the template. It works OK, but writing them are a really pain in the ass.

Azure Bicep is an abstraction over ARM templates, that aims to drastically simplify the pain  experience of writing ARM templates, it has a cleaner syntax, improved type safety, and better support for modularity and code re-use.   
Bicep code is transpiled to standard ARM Template JSON files.

For organizations starting a greenfield deployment only on Azure infrastructure and having no prior investment in other configuration languages such as Terraform, Bicep is a good option.

But the truth is that nowadays most of the companies I know that are working with Azure are using Terraform instead of Azure Bicep. 

Why is that? Terraform is a more mature option, works greats with multi-cloud scenarios or even when we want to provision resources on some PaaS/SaaS services, like: provisioning resources inside an Azure DevOps organization or a RabbitMq Cluster.   

Also, Azure Bicep is relatively new, so for quite some time the only viable options to work with IaC on Azure was either Terraform or ARM templates, so a lot of companies made the choice back then and right now there is not enough benefits to ditch Terraform for Bicep.


# Resources created by the bootstrap script

Here's a detailed list of which resources will be created during the bootstrap process:

- A Resource Group.
- An Storage Account that holds the Terraform State.
- A cannot-delete lock on the Storage Account. 
- A Service Principal that will be used by Azure Pipelines to deploy infrastructure onto Azure. This SP will have a custom role.
- A custom role used for deploying infrastructure. This role has the same permissions as ``Contributor`` but can create role assigmnemts. It also have permissions to read, write and delete data on Azure Key Vaults and App Configurations.
- A Key Vault to hold the credentials of the Service Principal. 
  - The script itself adds the SP credentials into the Key Vault, so you don't need to manipulate the Vault at all.
- A cannot-delete lock on the Key Vault. 
- An Azure DevOps variable group that holds the credentials of the Service Principal.
  - The bootstrap script links the Azure Key Vault we have created with this variable group and maps the SP credentials to the variable group.
- A second Service Principal with a ``Key Vault Administrator`` role associated at the resource group scope.


> _Why we need a second Service Principal?_

 To automate the deploying of infrastructure into Azure we will use Azure Pipelines, which means that the pipelines needs to retrieve the SP credentials to be able to deploy the resources.   
 
 And the SP credentials are stored in the Key Vault, so how we can the pipeline retrieve it?

There are a few options available:

- Create a variable group and store the SP credentials in plain text.
- Create a variable group, [link it with an existing Key Vault](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml#link-secrets-from-an-azure-key-vault) and use the secrets stored on the Vault as variables in the pipeline.
- Use the [Azure Key Vault Task](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault?view=azure-devops)


The first option is the easiest to implement but it is the less secure one, so it's discarded.

The other two options are both ok, use whatever you prefer, I personally prefer using a variable group because is less verbose that using the KeyVault task.   

Nevertheless, if you create a variable group and you want to link it to and existing Key Vault you need a Service Principal with permissions to retrieve secrets from the Vault.   
In this case I could use the Service Principal I'll be using to create the resources to Azure but it has a role with far too many permissions, so it is a better practice to create another Service Principal with only Key Vault permissions and use it only to link the variable group with the Vault.

# Bootstrap Script: Powershell part

The script is built using Powershell and the [Az module](https://docs.microsoft.com/es-es/powershell/azure/what-is-azure-powershell?view=azps-7.4.0) and it does the following steps:

- Creates a Resource Group and a Storage Account using the Azure Az Powershell Module.
- Inits Terraform using the Storage Account as backend.
- Imports those 2 resources into the Terraform state.
- Uses Terraform to create the rest of the resources listed on the previous section.

> _Why is the script using the Powershell Az Module only to create the resource group and the storage account? Why is it using Terraform to create the rest of the needed resources?_

We could do everything using purely scripting with Powershell and no using Terraform at all, but using Terraform to create the resources is simpler, less error prone and much more easy to mantain.   

Also if in a near future we want to update some of the existing resources or even add some additional ones it is easier to do it via Terraform than modifying the Powershell script.

The idea behind this script is that you don't need to touch the Powershell part of the script at all, in case you want to add some extra resources or modify the existing ones modify the ``main.tf`` file and execute the script. 

The next code snippet shows the entirety of the Powershell script:
```bash
Param(
    [Parameter(Mandatory=$True)]
    [bool] $ProvisionBootstrapResources = $false
)

function Set-ConfigVars
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $FileName
    )

    if (Test-Path -Path $FileName) 
    {
        $values = @{}
        Get-Content $FileName | Where-Object {$_.length -gt 0} | Where-Object {!$_.StartsWith("#")} | ForEach-Object {
            $var = $_.Split('=',2).Trim()
            $values[$var[0]] = $var[1]
        }
        return $values
    }
    else
    {
        Write-Error "Configuration file missing."
        exit 1
    }
}

function Connect-AzureSubscription
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $SubId,
        [Parameter(Mandatory=$true, Position=0)]
        [string] $TenantId
    )

    if (-not (Get-AzContext | Where-Object { $_.Subscription.Id -eq $SubId })) {
        Write-Host "Logging in to Azure Subscription..."
        Connect-AzAccount -SubscriptionId $SubId -TenantId $TenantId -ErrorAction Stop | Out-Null
    }

    Write-Host "Using Azure Subscription:" $(Get-AzContext).Subscription.Name -ForegroundColor Yellow
}

function New-ResourceGroup
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $RgName,
        [Parameter(Mandatory=$true, Position=1)]
        [string] $Region
    )

    if( -not (Get-AzResourceGroup -Name $RgName -ErrorAction SilentlyContinue))
    {
        Write-Host "Resource Group $RgName doesn't exist. Creating..."
        New-AzResourceGroup -Name $RgName -Location $Region -ErrorAction Stop
    }
    else
    {
        Write-Host "Resource Group $RgName already exists."
    }
}

function New-StorageAccount
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $RgName,
        [Parameter(Mandatory=$true, Position=1)]
        [string] $StAccName,
        [Parameter(Mandatory=$true, Position=2)]
        [string] $ContainerName,
        [Parameter(Mandatory=$true, Position=3)]
        [string] $Region
    )

    if( -not (Get-AzStorageAccount -Name $StAccName -ResourceGroupName $RgName -ErrorAction SilentlyContinue))
    {
        Write-Host "Storage Account $StAccName doesn't exist. Creating..."
        $storageAccount = New-AzStorageAccount -ResourceGroupName $RgName -Name $StAccName -Location $Region -SkuName Standard_LRS -ErrorAction Stop
        New-AzStorageContainer -Name $ContainerName -Permission Off -Context $storageAccount.Context -ErrorAction Stop
    }
    else
    {
        Write-Host "Storage Account $StAccName already exists."
        $storageAccount = Get-AzStorageAccount -Name $StAccName -ResourceGroupName $RgName -ErrorAction Stop
        If( -not (Get-AzStorageContainer -Name $ContainerName -Context $storageAccount.Context -ErrorAction SilentlyContinue))
        {
            Write-Host "Storage Container $ContainerName doesn't exist. Creating..."
            New-AzStorageContainer -Name $ContainerName -Permission Off -Context $storageAccount.Context -ErrorAction Stop
        }
        else
        {
            Write-Host "Storage Container $ContainerName already exists."
        }
    }
}

function Set-EnvVarsAsTfVars
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [hashtable] $ConfigVars
    )

    $env:TF_VAR_tf_state_resource_group_name=$ConfigVars.tf_state_resource_group_name
    $env:TF_VAR_tf_state_storage_account_name=$ConfigVars.tf_state_storage_account_name
    $env:TF_VAR_project_name=$ConfigVars.project_name
    $env:TF_VAR_azure_region=$ConfigVars.azure_region
    $env:TF_VAR_azdo_org_url=$ConfigVars.azdo_org_url
    $env:TF_VAR_azdo_project_name=$ConfigVars.azdo_project_name
    $env:TF_VAR_azdo_pat=$ConfigVars.azdo_pat
}

function Invoke-TerraformInit
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $RgName,
        [Parameter(Mandatory=$true, Position=1)]
        [string] $StAccName,
        [Parameter(Mandatory=$true, Position=2)]
        [string] $ContainerName
    )

    $result = & terraform init -input=false -backend=true -reconfigure `
        -backend-config="resource_group_name=$RgName" `
        -backend-config="storage_account_name=$StAccName" `
        -backend-config="container_name=$ContainerName" 2>&1 | out-string

    if ($result -notmatch "Terraform has been successfully initialized!" -eq $true) 
    {
        Write-Error $result
        exit 1
    }
    else
    {
        Write-Host "Terraform Initialized Successfully"
    }
}

function Import-TerraformState
{
    Param
    (
        [Parameter(Mandatory=$true, Position=0)]
        [string] $SubId,
        [Parameter(Mandatory=$true, Position=1)]
        [string] $RgName,
        [Parameter(Mandatory=$true, Position=2)]
        [string] $StAccName
    )

    Write-Host "Importing Resource Group to tf state" 
    $results1 = & terraform import "azurerm_resource_group.tf_state_rg" "/subscriptions/$SubId/resourceGroups/$RgName" 2>&1 | out-string

    Write-Host "Importing Storage Account to tf state"
    $results2 = & terraform import "azurerm_storage_account.tf_state_storage" "/subscriptions/$SubId/resourceGroups/$RgName/providers/Microsoft.Storage/storageAccounts/$StAccName" 2>&1 | out-string

    if (($results1 -notmatch "Resource already managed by Terraform") -and
        ($results1 -notmatch "Import successful!") -and
        ($results1 -notmatch "Cannot import non-existent remote object") -eq $true) {

        Write-Error $results1
        exit 1
    }

    if (($results2 -notmatch "Resource already managed by Terraform") -and
        ($results2 -notmatch "Import successful!") -and
        ($results2 -notmatch "Cannot import non-existent remote object") -eq $true) {

        Write-Error $results2
        exit 1
    }
}

function Invoke-TerraformPlan
{
    terraform plan
}

function Invoke-TerraformApply
{
    terraform apply -auto-approve
}

function main 
{
    $configVarsFileName = "config.env"

    $configVars = Set-ConfigVars -FileName $configVarsFileName
    
    Connect-AzureSubscription -SubId $configVars.azure_subscription_id `
                -TenantId $configVars.azure_tenant_id

    Set-EnvVarsAsTfVars -ConfigVars $configVars


    if ($ProvisionBootstrapResources -eq $true)
    {
        New-ResourceGroup -RgName $configVars.tf_state_resource_group_name `
                -Region $configVars.azure_region

        New-StorageAccount -RgName $configVars.tf_state_resource_group_name `
                -StAccName $configVars.tf_state_storage_account_name `
                -ContainerName $configVars.tf_state_storage_account_container_name `
                -Region $configVars.azure_region

        Invoke-TerraformInit -RgName $configVars.tf_state_resource_group_name `
                -StAccName $configVars.tf_state_storage_account_name `
                -ContainerName $configVars.tf_state_storage_account_container_name

        Import-TerraformState -SubId $configVars.azure_subscription_id `
                -RgName $configVars.tf_state_resource_group_name `
                -StAccName $configVars.tf_state_storage_account_name
    }

    if ($ProvisionBootstrapResources -eq $false)
    {
      
        Invoke-TerraformInit -RgName $configVars.tf_state_resource_group_name `
                    -StAccName $configVars.tf_state_storage_account_name `
                    -ContainerName $configVars.tf_state_storage_account_container_name
    }
    
    Invoke-TerraformPlan
    Read-Host -Prompt "Press any key to run terraform apply or CTRL+C to quit" 
    
    Invoke-TerraformApply
}

main
```
The script is pretty self-explanatory, but nonetheless here's a quick summary explaining what every function does.

- ``Set-ConfigVars``: Gets the script configuration. _More info about the configuration on the "Script configuration" section below._ 
- ``Connect-AzureSubscription``:  Connects to the specified Azure subscription.
- ``Set-EnvVarsAsTfVars``: The ``Set-ConfigVars`` function gets the script configuration and this one stores the config variables as [Terraform TF_VAR_ environment variables](https://www.terraform.io/cli/config/environment-variables#tf_var_name) so the config can be used by the Terraform part of the script.
- ``New-ResourceGroup``: Creates a Resource Group, if it doesn't exist.
- ``New-StorageAccount``: Creates a Storage Account and a Storage Container, if they don't exist.
- ``Invoke-TerraformInit``: Initializes Terraform using the Storage Container as backend.
- ``Import-TerraformState``: Adds the Storage Account and the Resource Container into the Tf state, so any future changes can be tracked using the ``Terraform Plan`` command.
- ``Invoke-TerraformPlan``: Runs the ``Terraform Plan`` command.
- ``Invoke-TerraformApply``:  Runs the ``Terraform Apply`` command.

To know the use of the ``$ProvisionBootstrapResources`` parameter and how to execute the script, read the _"How to run the script"_ section below.


# Bootstrap script: Terraform part

The Powershell script only creates the necessary resources to start using Terraform (Resource Group + Storage Account), the rest of the resources are created via Terraform.

The next code snippet shows the terraform part of the bootstrap script.

```csharp
terraform {

  backend "azurerm" {
    key = "bootstrap.tfstate"
  }

  required_providers {

    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">=3.0.0"
    }

    azuredevops = {
      source = "microsoft/azuredevops"
      version = ">=0.2.0"
    }

    azuread = {
      source  = "azuread"
      version = ">=2.20.0"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "azuredevops" {
  org_service_url       = var.azdo_org_url
  personal_access_token = var.azdo_pat
}

provider "azuread"{
}

## Get the configuration of the AzureRM provider
data "azurerm_client_config" "current" {}


## Get the AzDo Team Project 
data "azuredevops_project" "project" {
  name = var.azdo_project_name
}

##########################################################################################
## Start Importing existing resources into tf
##########################################################################################

## Create resource group. Already exists created by azure-bootstrap-terraform-init.sh
resource "azurerm_resource_group" "tf_state_rg" {
  name     = var.tf_state_resource_group_name
  location = var.azure_region
  tags = var.default_tags
}

## Creates store account that hold Terraform shared state. Already exists created by azure-bootstrap-terraform-init.sh
resource "azurerm_storage_account" "tf_state_storage" {
  name                     = var.tf_state_storage_account_name
  resource_group_name      = azurerm_resource_group.tf_state_rg.name
  location                 = azurerm_resource_group.tf_state_rg.location
  account_tier             = "Standard"
  account_kind             = "StorageV2"
  account_replication_type = "LRS"
  
  tags = merge( 
    var.default_tags,
    {
      "description" = "Storage Account that holds the Terraform state files."
    })
}

## Lock the storage account. It cannot be deleted because it is needed by Terraform.
resource "azurerm_management_lock" "lock_tf_storage_account" {
  name       = "lock-bs-tf-stacct-${var.project_name}"
  scope      = azurerm_storage_account.tf_state_storage.id
  lock_level = "CanNotDelete"
  notes      = "Locked because it's needed by Terraform"
}

##########################################################################################
## End Importing existing resources into tf
##########################################################################################

##########################################################################################
## Start Creating KeyVault to hold SP credentials
##########################################################################################

## KeyVault to hold SP creds
resource "azurerm_key_vault" "sp_creds_kv" {
  name                        = "kv-bs-tf-${var.project_name}"
  location                    = azurerm_resource_group.tf_state_rg.location
  resource_group_name         = azurerm_resource_group.tf_state_rg.name
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "standard"
  soft_delete_retention_days  = 15
  enable_rbac_authorization   = true
  purge_protection_enabled    = false
  tags                        = merge( 
    var.default_tags,
    {
      "description" = "KeyVault that holds the SP credentials for deploying infrastructure"
    })
}

## Lock the key vault. It cannot be deleted because it is needed by Azure DevOps
resource "azurerm_management_lock" "lock_sp_kv" {
  name       = "lock-bs-tf-kv-${var.project_name}"
  scope      = azurerm_key_vault.sp_creds_kv.id
  lock_level = "CanNotDelete"
  notes      = "Locked because it's needed by Azure DevOps"
}

## Add myself as a KV Admin role. This assignment is required to later add the IaC SP credentials into the KV
resource "azurerm_role_assignment" "me_keyvault_role" {
  scope                            = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${azurerm_resource_group.tf_state_rg.name}"
  role_definition_name             = "Key Vault Administrator"
  principal_id                     = data.azurerm_client_config.current.object_id
}

##########################################################################################
## End Creating KeyVault to hold SP credentials
##########################################################################################

#########################################################################################
## Start creating SP to be used by Azure DevOps variable group  to access the Key Vault
##########################################################################################

## Create an AAD application, it's needed to create a SP
resource "azuread_application" "azdo_keyvault_app" {
  display_name = "app-bs-tf-azdo-vargroup-kv-connection-${var.project_name}"
}

## Create an AAD Service Principal
resource "azuread_service_principal" "azdo_keyvault_sp" {
  application_id = azuread_application.azdo_keyvault_app.application_id
}

## Creates a password for the AAD app
resource "azuread_application_password" "azdo_keyvault_sp_password" {
  application_object_id = azuread_application.azdo_keyvault_app.id
  display_name          = "TF generated password" 
  end_date              = "2040-01-01T00:00:00Z"
}

## Assign a KV Admin role to the SP. The role is assigned at resource group scope
resource "azurerm_role_assignment" "azdo_keyvault_role" {
  scope                            = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${azurerm_resource_group.tf_state_rg.name}"
  role_definition_name             = "Key Vault Administrator"
  principal_id                     = azuread_service_principal.azdo_keyvault_sp.id
  skip_service_principal_aad_check = true
}

## Create a Azure DevOps Service Endpoint to access to KV
resource "azuredevops_serviceendpoint_azurerm" "keyvault_access" {
  project_id            = data.azuredevops_project.project.id
  service_endpoint_name = "service-endpoint-bs-tf-azdo-vargroup-kv-connection-${var.project_name}"
  credentials {
    serviceprincipalid  = azuread_application.azdo_keyvault_app.application_id
    serviceprincipalkey = azuread_application_password.azdo_keyvault_sp_password.value
  }
  azurerm_spn_tenantid      = data.azurerm_client_config.current.tenant_id
  azurerm_subscription_id   = data.azurerm_client_config.current.subscription_id
  azurerm_subscription_name = "Management Subscription"
}
##########################################################################################
## End creating SP to be used by Azure DevOps variable group  to access the Key Vault
##########################################################################################

##########################################################################################
## Start creating SP to be used by AzDo Pipelines to deploy infrastructure to Azure
##########################################################################################

## Create an AAD application, it's needed to create a SP
resource "azuread_application" "iac_app" {
  display_name = "app-bs-tf-deploy-iac-azdo-pipelines-${var.project_name}"
}

## Create an AAD Service Principal
resource "azuread_service_principal" "iac_sp" {
  application_id = azuread_application.iac_app.application_id
}

## Creates a random password for the AAD app
resource "azuread_application_password" "iac_sp_password" {
  application_object_id = azuread_application.iac_app.id
  display_name          = "TF generated password"   
  end_date              = "2040-01-01T00:00:00Z"
}

# Create a custom role for this SP
resource "azurerm_role_definition" "iac_custom_role" {
  name        = "role-iac-deploy-${var.project_name}"
  scope       = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  description = "This is a custom role created via Terraform. It has the same permissions as Contributor but can create role assigmnemts. It also have permissions to read, write and delete data on  Azure Key Vault and App Configuration."
  permissions {
    actions     = ["*"]
    not_actions = [
      "Microsoft.Authorization/elevateAccess/Action",
      "Microsoft.Blueprint/blueprintAssignments/write",
      "Microsoft.Blueprint/blueprintAssignments/delete",
      "Microsoft.Compute/galleries/share/action"
    ]
    data_actions = [ 
      "Microsoft.KeyVault/vaults/*",
      "Microsoft.AppConfiguration/configurationStores/*/read",
      "Microsoft.AppConfiguration/configurationStores/*/write",
      "Microsoft.AppConfiguration/configurationStores/*/delete"
    ]
    not_data_actions = []
  }
  assignable_scopes = [
    "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  ]
}

## Assign the custom role to the SP. The role is assigned at subscription scope.
resource "azurerm_role_assignment" "iac_role_assignment" {
  scope                            = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  role_definition_name             = "role-iac-deploy-${var.project_name}"
  principal_id                     = azuread_service_principal.iac_sp.id
  skip_service_principal_aad_check = true
  depends_on = [
    azurerm_role_definition.iac_custom_role
  ]
}

## Store SP client secret in the KV
resource "azurerm_key_vault_secret" "iac_sp_secret" {
  name         = "sp-bs-tf-iac-client-secret"
  value        = azuread_application_password.iac_sp_password.value
  key_vault_id = azurerm_key_vault.sp_creds_kv.id
  tags = var.default_tags
}

## Store SP client secret in the KV
resource "azurerm_key_vault_secret" "iac_sp_clientid" {
  name         = "sp-bs-tf-iac-client-id"
  value        = azuread_service_principal.iac_sp.application_id
  key_vault_id = azurerm_key_vault.sp_creds_kv.id
  tags = var.default_tags
}

## Store SP client secret in the KV
resource "azurerm_key_vault_secret" "iac_sp_tenant" {
  name         = "sp-bs-tf-iac-tenant-id"
  value        = data.azurerm_client_config.current.tenant_id
  key_vault_id = azurerm_key_vault.sp_creds_kv.id
  tags = var.default_tags
}

## Store SP client secret in the KV
resource "azurerm_key_vault_secret" "iac_sp_subid" {
  name         = "sp-bs-tf-iac-subscription-id"
  value        = data.azurerm_client_config.current.subscription_id
  key_vault_id = azurerm_key_vault.sp_creds_kv.id
  tags = var.default_tags
}

##########################################################################################
## End creating SP to be used by AzDo Pipelines to deploy infrastructure to Azure
##########################################################################################

#########################################################################################
## Start creating Azure DevOps variable Group used for deploy IaC
##########################################################################################

## Create AZDO variable group with IaC SP credentials
resource "azuredevops_variable_group" "azdo_iac_var_group" {
  project_id   = data.azuredevops_project.project.id
  name         = "vargroup-bs-tf-iac-${var.project_name}"
  allow_access = true

  key_vault {
    name                = azurerm_key_vault.sp_creds_kv.name
    service_endpoint_id = azuredevops_serviceendpoint_azurerm.keyvault_access.id
  }

  depends_on = [
    azurerm_key_vault_secret.iac_sp_secret,
    azurerm_key_vault_secret.iac_sp_clientid,
    azurerm_key_vault_secret.iac_sp_tenant,
    azurerm_key_vault_secret.iac_sp_subid
  ]

  variable {
    name = "sp-bs-tf-iac-client-id"
  }

  variable {
    name = "sp-bs-tf-iac-client-secret"
  }

  variable {
    name = "sp-bs-tf-iac-tenant-id"
  }

  variable {
    name = "sp-bs-tf-iac-subscription-id"
  }
}

##########################################################################################
## End creating Azure DevOps variable Group used for deploy IaC
##########################################################################################
```
 As I did with the Powershell part of the script, let me run another quick summary explaining what Terraform creates.

The first two resources you'll see in the ``main.tf`` file are these ones:

```csharp
## Create resource group. Already exists created by azure-bootstrap-terraform-init.sh
resource "azurerm_resource_group" "tf_state_rg" {
  name     = var.tf_state_resource_group_name
  location = var.azure_region
  tags = var.default_tags
}

## Creates store account that hold Terraform shared state. Already exists created by azure-bootstrap-terraform-init.sh
resource "azurerm_storage_account" "tf_state_storage" {
  name                     = var.tf_state_storage_account_name
  resource_group_name      = azurerm_resource_group.tf_state_rg.name
  location                 = azurerm_resource_group.tf_state_rg.location
  account_tier             = "Standard"
  account_kind             = "StorageV2"
  account_replication_type = "LRS"
  
  tags = merge( 
    var.default_tags,
    {
      "description" = "Storage Account that holds the Terraform state files."
    })
}
```
Terraform is **NOT** creating a new Resource Group and Storage Account.   
On the Powershell part of the script we imported these resources into the Tf state, here we're simply declaring them so we can keep track of them via Terraform.

 In the ``main.tf`` file we're using those 3 providers:

 - The [azuread](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs) provider.
 - The [azuredevops](https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs) provider.
 - The [azurerm](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) provider.


With the azuread provider we create the following resources:
  - A Service Principal that will be used to deploy infrastructure.
  - A Service Principal that will be used to link the Azure DevOps variable group with the Vault.

With the azuredevops provider we create the following resources:
  - A Service Endpoint _(is required to link the variable group with the Vault and makes use of the second SP)_.
  - A Variable Group linked with the Vault _(to link it, it uses the Service Endpoint)_.

With the azurerm provider we create the following resources:
  - An Storage Account cannot-delete Lock.
  - An Azure KeyVault.
  - An Azure KeyVault cannot-delete Lock.
  - Assign myself a "Key Vault Administrator" Role _(this role assignment is necessary because later on the script we are going to add the SP credentials into the Vault)_.
  - Create a custom role _(it will be used to deploy the infrastructure)_ and assign it to the SP.
  - Assign a "Key Vault Administrator" Role to the second SP _(this role assignment is necesarry to link the variable group with the Vault)_.
  - Store the SP credentials into the Vault.

Also, if you take a look at my [GitHub repository](https://github.com/karlospn/bootstrapping-azure-subscription-and-azdo-project-for-terraform) you'll see that there is a ``variables.tf`` file. More info about it in the next section.


# Script configuration

Reusability is key, I don't want to modify the script every time I need to bootstrap a new subscription.

To avoid that, there is a ``config.env`` file that contains the script configuration.

The configuration variables are used in the Powershell script and also in the Terraform files. The script stores the configuration variables as [Terraform TF_VAR_ environment variables](https://www.terraform.io/cli/config/environment-variables#tf_var_name) so they can be used within the ``main.tf`` file.   

> You can change the values on the ``config.env`` file to your liking, but you must **NOT** change the name of the variables or the script will break.

The config variables are the following ones:

- ``tf_state_resource_group_name``: The name of the resource group.
- ``tf_state_storage_account_name``: The name of the storage account.
- ``tf_state_storage_account_container_name``: The name of the storage account container.
- ``project_name``: The name of the project. It will be added as a suffix in all the created resources.
- ``azure_region``: The azure region where all the resources will be created.
- ``azure_subscription_id``: The azure subscription ID.
- ``azure_tenant_id``: The azure tenant ID.
- ``azdo_org_url``: The URL of the Azure DevOps organization.
- ``azdo_project_name``: The name of the Azure DevOps project where the variable group will be created.
- ``azdo_pat``: An Azure DevOps PAT (Personal Access Token).

Example:
```bash
## Terraform State Variable
tf_state_resource_group_name=rg-bs-tf-myproject-dev
tf_state_storage_account_name=stbstfmyprojectdev
tf_state_storage_account_container_name=tfstate

## Project Name
project_name=myproject-dev

## Azure Variables
azure_region=westeurope
azure_subscription_id=c179c52f-af4d-4a1a-adbe-2a27d480c62d
azure_tenant_id=da0d66e4-f338-454c-b0e5-cbdbf4fc385f

## Azure DevOps Variables
azdo_org_url=https://dev.azure.com/cponsn
azdo_project_name=demos
azdo_pat=12p3j12p31290j213021asdpsdj
```
# How to run the script

**To run it you need to set a parameter named: ``ProvisionBootStrapResources``.** 

- _Example: ``./Initialize-AzureBootstrapProcessForTerraform.ps1 -ProvisionBootStrapResources $True``_

When the ``ProvisionBootStrapResources`` parameter is set to ``$True`` it will execute the entire script, which means:
- Creating the resource group and the storage account for the tf state using the Powershell Az module.
- Import them into the Tf state.
- Executing the Terraform Init, Plan and Apply commands to create the rest of the resources.      

**If this is the first time you run the script and want to create the all the resources from zero, set it to ``$True``.**

When the ``ProvisionBootStrapResources`` parameter is set to ``$False`` it will skip the steps of creating the resource group and the storage account and it will only run the Terraform Init, Plan and Apply steps.   
**If you already ran the bootstrap script previously and you have modified the ``main.tf`` file to add or update some existing resources set it to ``$False``.**

# Where to run the script

There are 2 options available, run it on your **local machine** or on **Azure Cloud Shell**.

To run the script on your **local machine** you'll need to have already installed:
-  Terraform 
-  Azure Powershell Az Module. 
   -  For more information go to: https://docs.microsoft.com/es-es/powershell/azure/what-is-azure-powershell?view=azps-7.4.0


To run the script on **Azure Cloud Shell** you'll need to upload the following files into the remote workspace:

- ``Initialize-AzureBootstrapProcessForTerraform.ps1``
- ``config.env``
- ``main.tf``
- ``variables.tf``

And afterwards, just execute the ``Initialize-AzureBootstrapProcessForTerraform.ps1`` script.

![tf-bs-azdo-cloudshell](/img/tf-bs-azdo-cloudshell.png)

# Permissions needed to execute the script

To run the script you'll need to have the following permissions:

- An ``Owner`` Role on the target Azure Subscription.
- An ``Application Administrator`` Role on Azure Active Directory.
- An ``Azure DevOps PAT`` (Personal Access Token) with a Full Access scope.


# Azure DevOps IaC pipeline example

After executing the script we're ready to start deploying infrastructure to Azure using Azure Pipelines.    

I have created an example pipeline to show you how it looks.   
The pipeline has 3 runtime parameters. The Runtime parameters let you have more control over what values can be passed to a pipeline. In our case we are defining which Terraform commands should the pipeline execute.

![tf-bs-azdo-pipelines-run-pipeline](/img/tf-bs-azdo-pipelines-run-pipeline.png)

The pipeline uses the variable group we have created to obtain the credentials of the SP stored on the Key Vault.

![tf-bs-azdo-variable-group](/img/tf-bs-azdo-variable-group.png)

The next code snippet is an example of a pipeline that does exactly that.

```yaml
trigger: none

parameters:
- name: terraform_destroy
  type: boolean
  default: false
- name: terraform_apply
  type: boolean
  default: false
- name: terraform_plan
  type: boolean
  default: true


variables:
  - group: vargroup-bs-tf-iac-myproject-dev
  - name: STATE_RESGRP
    value: rg-bs-tf-myproject-dev
  - name: STATE_ACCOUNT
    value: stbstfmyprojectdev
  - name: STATE_CONTAINER
    value: tfstate
  - name: KEY_NAME
    value: shared-svc-group-myproject-dev
  - name: CURRENT_PATH
    value: ./samples/azure-pipelines

pool: 
  vmImage: 'ubuntu-latest'

steps:  
  - bash: |
      terraform init -input=false -backend=true -reconfigure \
      -backend-config="resource_group_name=$(STATE_RESGRP)" \
      -backend-config="storage_account_name=$(STATE_ACCOUNT)" \
      -backend-config="container_name=$(STATE_CONTAINER)" \
      -backend-config="key=$(KEY_NAME).tfstate"
    workingDirectory: $(CURRENT_PATH)
    displayName: Initialize Terraform backend state
    env:
      ARM_CLIENT_ID: $(sp-bs-tf-iac-client-id)
      ARM_CLIENT_SECRET: $(sp-bs-tf-iac-client-secret)
      ARM_TENANT_ID: $(sp-bs-tf-iac-tenant-id)
      ARM_SUBSCRIPTION_ID: $(sp-bs-tf-iac-subscription-id)
  
  - bash: |
      terraform plan -input=false
    condition: and(succeeded(), eq('${{ parameters.terraform_plan }}', true))
    workingDirectory: $(CURRENT_PATH)
    displayName: Plan Terraform changes
    env:
      ARM_CLIENT_ID: $(sp-bs-tf-iac-client-id)
      ARM_CLIENT_SECRET: $(sp-bs-tf-iac-client-secret)
      ARM_TENANT_ID: $(sp-bs-tf-iac-tenant-id)
      ARM_SUBSCRIPTION_ID: $(sp-bs-tf-iac-subscription-id)
  
  - bash: |
      terraform apply -input=false -auto-approve
    condition: and(succeeded(), eq('${{ parameters.terraform_apply }}', true))
    workingDirectory: $(CURRENT_PATH)
    displayName: Apply Terraform changes
    env:
      ARM_CLIENT_ID: $(sp-bs-tf-iac-client-id)
      ARM_CLIENT_SECRET: $(sp-bs-tf-iac-client-secret)
      ARM_TENANT_ID: $(sp-bs-tf-iac-tenant-id)
      ARM_SUBSCRIPTION_ID: $(sp-bs-tf-iac-subscription-id)
  - bash: |
      terraform destroy -input=false -auto-approve
    condition: and(succeeded(), eq('${{ parameters.terraform_destroy }}', true))
    workingDirectory: $(CURRENT_PATH)
    displayName: Destroy Terraform 
    env:
      ARM_CLIENT_ID: $(sp-bs-tf-iac-client-id)
      ARM_CLIENT_SECRET: $(sp-bs-tf-iac-client-secret)
      ARM_TENANT_ID: $(sp-bs-tf-iac-tenant-id)
      ARM_SUBSCRIPTION_ID: $(sp-bs-tf-iac-subscription-id)
```
And here's an example of how the output of the pipeline looks like:

![tf-bs-azdo-pipeline-tf-plan](/img/tf-bs-azdo-pipeline-tf-plan.png)


# Reference links

Whenever I use some code that is taken from somewhere else or I draw inspiration from other person work I like to reference it in my posts.   

In this case I haven taken a few ideas from this project, so kudos to him.

- https://github.com/benc-uk/terraform-mgmt-bootstrap



