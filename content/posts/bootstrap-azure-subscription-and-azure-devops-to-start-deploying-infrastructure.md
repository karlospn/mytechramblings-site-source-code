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

In a nutshell, infrastructure as code (IaC) is the process of managing and provisioning computer data centers through machine-readable definition files, instead of through physical hardware configuration or interactive configuration tools.

Using IaC on Azure with Azure DevOps requires of a minimal bootstrap process, you could do it manually and it will work. The required steps are the following ones:
-  Create an Storage Account to store the Terraform state
-  Create a Service Principal on our AAD with a role with enough permissions to create new resources on Azure.
-  Store the Service Principal credential somewhere safe.
-  On Azure Pipelines create a CI/CD pipeline that gets the Service Principal credentials and deploys the infrastructure on Azure using Terraform.

The steps required are quite simple and as I stated before you could do it manually, but you'll have to do them every time you want to deploy resources into a new subscription. So, having some kind of automation seems the way to go here.

And that's what I want to show you in this post, how to programmatically bootstrap an Azure subscription and an Azure DevOps project to start deploying Infrastructure as Code.

# Azure Bicep vs ARM Templates vs Terraform

When working with Azure we have 2 native options for IaC: Azure Resource Manager (ARM) templates or Azure Bicep.

ARM templates are essentially just JSON files with a few extended capabilities, describing the expected infrastructure to end up with, after applying the template. It works good, but writing them are a really pain in the ass.

Azure Bicep is an abstraction over ARM templates, that aims to drastically simplify the pain  experience of writing ARM templates, it has a cleaner syntax, improved type safety, and better support for modularity and code re-use.   
Bicep code is transpiled to standard ARM Template JSON files.

For organizations starting a greenfield deployment **ONLY** on Azure infrastructure and having no prior investment in other configuration languages such as Terraform, Bicep would be a great option.

But the truth is that nowadays most of the companies I know that are working with Azure are using Terraform instead of Azure Bicep. 

Why is that?  Terraform is a more mature option, that works greats with multi-cloud scenarios or even when we want to provision things on some PaaS/SaaS services, like: provisioning things on an Azure DevOps organization or a RabbitMq Cluster hosted wherever, etc.   

Also, Azure Bicep is relatively new, so for quite some time the only viable option to work with IaC on Azure was either Terraform or ARM templates, so a lot of companies made the choice back then and right now there is not enough benefits to ditch Terraform for Bicep.





# Bootstrap Phase







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

![vs-dump-parallel-stacks](/img/vs-dump-parallel-stacks.png)


# Reference links

Whenever I use some code that is taken from somewhere else or I draw inspiration from other person work I like to reference it in my posts.   

In this case I haven taken a few ideas from this project, so kudos to him.

- https://github.com/benc-uk/terraform-mgmt-bootstrap



