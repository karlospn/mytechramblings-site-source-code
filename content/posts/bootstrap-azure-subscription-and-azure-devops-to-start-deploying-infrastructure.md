---
title: "How to bootstrap Terraform and Azure DevOps to start deploying your infrastructure as code to Azure"
date: 2022-04-15T14:22:10+02:00
tags: ["azure", "devops", "terraform", "iac"]
description: "TBD"
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/bootstrapping-azure-subscription-and-azdo-project-for-terraform).

Nowadays almost every developer knows what infrastructure as code is.

In a nutshell, infrastructure as code (IaC) is the process of managing and provisioning computer resources through machine-readable definition files, instead of through physical hardware configuration or interactive configuration tools.

Using IaC on Azure with Azure DevOps requires a minimal bootstrap process. The required steps are the following ones:
-  Create a Resource Group.
-  Create an Storage Account to store the Terraform state.
-  Create a Service Principal on our AAD.
-  Assign a role to the SP with enough permissions to create new resources on Azure.
-  Store the SP credentials somewhere safe.
-  On Azure Pipelines create a CI/CD pipeline that gets the SP credentials and using Terraform deploys the infrastructure on Azure.

As you can see the required steps are quite simple and you could do them manually, but you'll have to do them every time you want to start deploying resources into a new subscription. So, having some kind of automation seems the way to go here.

And that's what I want to show you in this post, how to programmatically bootstrap an Azure subscription and an Azure DevOps project to start deploying Infrastructure as Code.

# Azure Bicep vs ARM Templates vs Terraform

First of all let me talk a little about which tools are available when working with IaC on Azure.

When working with Azure we have 2 native options for IaC: Azure Resource Manager (ARM) templates or Azure Bicep.

ARM templates are essentially just JSON files with a few extended capabilities, describing the expected infrastructure to end up with, after applying the template. It works good, but writing them are a really pain in the ass.

Azure Bicep is an abstraction over ARM templates, that aims to drastically simplify the pain  experience of writing ARM templates, it has a cleaner syntax, improved type safety, and better support for modularity and code re-use.   
Bicep code is transpiled to standard ARM Template JSON files.

For organizations starting a greenfield deployment **ONLY** on Azure infrastructure and having no prior investment in other configuration languages such as Terraform, Bicep would be a great option.

But the truth is that nowadays most of the companies I know that are working with Azure are using Terraform instead of Azure Bicep. 

Why is that?  Terraform is a more mature option, that works greats with multi-cloud scenarios or even when we want to provision things on some PaaS/SaaS services, like: provisioning things on an Azure DevOps organization or a RabbitMq Cluster hosted wherever, etc.   

Also, Azure Bicep is relatively new, so for quite some time the only viable option to work with IaC on Azure was either Terraform or ARM templates, so a lot of companies made the choice back then and right now there is not enough benefits to ditch Terraform for Bicep.


# Resources created during the bootstrap process

Here's a detailed list of which resources will be created during the bootstrap process:

- A Resource Group.
- An Storage Account to hold the Terraform State.
- A Service Principal that will be used by Azure DevOps to deploy the infrastructure onto Azure. This SP uses a custom role.
- A custom role. 
  - This custom role has the same permissions as ``Contributor`` but can create role assigmnemts. It also have permissions to read, write and delete data on Azure Key Vault and App Configuration.
- A Key Vault to hold the credentials of the Service Principal.
  - The script by itself adds the SP credentials into the KV, you don't need to do anything.
- An Azure DevOps variable group that holds the credentials of the Service Principal.
  - The script links the Azure Key Vault we have created with this variable group and map the SP credentials to the variable group, but to do that we will need another Service Principal.
- A secondary Service Principal with a ``Key Vault Administrator`` role associated at the resource group scope.

# Bootstrap Script

The script is built using Powershell and the Az module and it does the following steps:

- Creates a Resource Group and a Storage Account using the Azure Az Powershell Module.
- Inits Terraform using the Storage Account as backend.
- Imports those 2 resources into the Terraform state.
- Uses Terraform to create the rest of the resources.

_Why is the script using the Powershell Az Module only to create the resource group and the storage account? Why is it using Terraform to create the rest of the needed resources?_

We could do everything using purely scripting with Powershell and no using Terraform at all, but using Terraform to create the needed resources in this script is simpler, less error prone and much more easy to mantain.   

Also if in a near future we want to update some of the existing resources or even add some additional ones it is easier to do it via Terraform than modifying the Powershell script.

The idea behind this script is that you don't need to touch the Powershell script at all, in case you want to add some extra resources or modify the existing ones modify the ``main.tf`` file. 

The next code snippet show the entirety of the script:
```powershell
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

# Script configuration

Reusability is key, I don't want to modify the script every time I need to bootstrap a new subscription.

To avoid that, there is a ``config.env`` file that contains the script configuration.

You can change the values on this file to your liking, but you must **NOT** change the name of the variables within the ``config.env`` file or the script will break.


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

**To run it you need to pass it a parameter named: ``ProvisionBootStrapResources``.** 

- _Example: ``./Initialize-AzureBootstrapProcessForTerraform.ps1 -ProvisionBootStrapResources $True``_

When the ``ProvisionBootStrapResources`` parameter is set to ``$True`` it will execute the entire script, which means:
- Creating the resource group and the storage account for the tf state using the Powershell Az module
- Executing the Terraform Init, Plan and Apply commands to create the rest of the resources.      

**If this is the first time you run the script and want to create the all the resources from zero, set it to ``$True``.**

When the ``ProvisionBootStrapResources`` parameter is set to ``$False`` it will skip the steps of creating the resource group and the storage account, it will only run the Terraform Init, Plan and Apply steps.   
**If you have modified the ``main.tf`` file to add or update some existing resources set it to ``$False``.**

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

![vs-dump-parallel-stacks](/img/vs-dump-parallel-stacks.png)

# Permissions needed to execute the script

To run the script you'll need to have the following permissions:

- An ``Owner`` Role on the target Azure Subscription.
- An ``Application Administrator`` Role on Azure Active Directory.
- An ``Azure DevOps PAT`` (Personal Access Token) with a Full Access scope.


# Azure DevOps pipeline example

After executing the script we're ready to start deploying infrastructure to Azure using Azure Pipelines.    

The next code snippet is an example of a pipeline that does exactly that.

The pipeline has 3 runtime parameters. The Runtime parameters let you have more control over what values can be passed to a pipeline. In our case we are defining which Terraform commands should the pipeline execute.

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

# Reference links

Whenever I use some code that is taken from somewhere else or I draw inspiration from other person work I like to reference it in my posts.   

In this case I haven taken a few ideas from this project, so kudos to him.

- https://github.com/benc-uk/terraform-mgmt-bootstrap



