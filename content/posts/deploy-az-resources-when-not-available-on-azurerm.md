---
title: "How to deploy an Azure resource using Terraform when it is not available in the AzureRM official provider"
date: 2022-11-03T23:10:40+01:00
tags: ["azure", "devops", "iac", "terraform", "cloud"]
description: "This post is going to walk you through the options available when we want to create or update a service on Azure using Terraform, but it is not available on the AzureRM Terraform provider."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/deploy-az-resource-using-terraform-when-not-available-on-azurerm-provider).

Everyone who has worked long enough with Terraform in Azure has been in the position of wanting to deploy a resource that's not available on the official [Azure Terraform provider](https://registry.terraform.io/providers/hashicorp/azurerm).   

The same situation also happens when trying to enable a feature of an existing resource and that feature is missing from the AzureRM Terraform provider.

A solution to those problems might be to switch to Bicep or Azure ARM Templates, but if all my cloud infrastructure is written as code using Terraform, why should I switch to another tool? What can I do to keep using Terraform?

Well, that's the point of this post, **to review which options we have available when we want to create or update a service on Azure using Terraform, but it is not available on the AzureRM Terraform provider.**

Before we dive further into this post, let me remind you that if you can avoid using any of the options we're going to discuss in this post then do so.    
Doing what we're going to discuss here should be a last resort and it makes sense only **when the AzureRM provider doesn't have an implementation for the service we want to create or update**.

# **Available options for creating or updating an Azure resource when it is not available in the AzureRM provider**

Nowadays there are **3 options available**:
- Using the  ``null_resource`` alongside with the ``local-exec`` provisioner to execute a script that uses the ``Azure CLI`` or the ``Azure Az Powershell Module``.
- Using the ``azurerm_resource_group_template_deployment`` from the AzureRM provider to deploy an ``ARM Template``.
- Using the ``AzAPI`` provider.

In the following sections we're going to talk about benefits and downsides for all the above options.

# **Using the null_resource + local-exec provisioner + Azure CLI or Azure Az Powershell Module**

> **Important**: Use this approach as a **last resort**. The other options discussed in this post are a far better alternative.

The  ``null_resource``  is a Terraform resource that doesn't do anything.   
If you need to run provisioners that aren't directly associated with a specific resource, you can associate them with a ``null_resource``.  

The ``local-exec`` provisioner is used to invoke a local executable on the machine running Terraform.

Pairing the ``null_resource``  with the ``local-exec`` provisioner allows us to define a Terraform resource that can run a local executable on our local machine, such as an external script that uses the Azure CLI to create a new resource on Azure.

The next code snippet shows an example of how you could execute a script that creates an Azure Resource Group using  the ``null_resource`` and the ``local-exec`` provisioner.

```csharp
resource "null_resource" "res_group" {
    triggers = {
        location = "westeurope"
    }
    provisioner "local-exec" {
        command = "./create_resource_group.sh ${self.triggers.location}" 
        interpreter = ["bash", "-c"]
    }
}
```

The ``triggers`` argument allows specifying an arbitrary set of values that, when changed, will cause the resource to be replaced.

The ``create_resource_group.sh`` script will look like this:

```bash
#!/bin/bash
LOCATION="$1"
az group create -l $LOCATION -n MyResourceGroup
```


## **How to destroy a resource using the local-exec provisioner**

By default, the ``local-exec`` provisioner will run only when the resource they are defined within is created. It only runs during resource creation, not during updating or any other lifecycle.

If we want that our resource gets deleted using the ``local-exec`` provisioner we need to use the ``when`` argument. If  ``when = destroy`` argument is specified, the provisioner will run when the resource is destroyed.

The next code snippet shows an example of how you could use the ``null_resource`` and the ``local-exec`` provisioner to run a script that creates an Azure Resource Group and another one that deletes it.

```csharp
resource "null_resource" "res_group" {
    count = 0
    triggers = {
        location = "westeurope"
    }
    provisioner "local-exec" {
        command = "./create_resource_group.sh ${self.triggers.location}" 
        interpreter = ["bash", "-c"]
    }

    provisioner "local-exec" {
        when = destroy
        command = "./destroy_resource_group.sh"
        interpreter = ["bash", "-c"]
    }
}
```
The ``destroy_resource_group.sh`` script will look like this:

```bash
#!/bin/bash
az group delete -n MyResourceGroup --yes
```
There is one big problem with this approach.   

When we remove a resource from the Terraform file and execute the ``apply`` command normally the resource gets deleted from the Terraform state file and from Azure, that's the common behaviour of Terraform, but it doesn't apply to the ``local-exec`` provisioner.

If a  ``null_resource`` block with a ``local-exec`` provisioner  gets removed entirely from the Terraform file the resource **won't be destroyed** at all. To work around this, a multi-step process needs to be used to safely remove a resource:
   - Update the resource configuration to include count = 0.
   - Apply the configuration to destroy any existing instances of the resource, including running the destroy provisioner.
   - Remove the resource block entirely from configuration, along with its provisioner blocks.
   - Apply again, at which point no further action should be taken since the resources were already destroyed.

The next code snippet shows an example of how you could destroy an Azure Resource Group that's been created using the ``null_resource`` and the ``local-exec`` provisioner.

```csharp
resource "null_resource" "res_group" {
    count = 0
    triggers = {
        location = "westeurope"
    }
    provisioner "local-exec" {
        command = "./create_resource_group.sh ${self.triggers.location}" 
        interpreter = ["bash", "-c"]
    }

    provisioner "local-exec" {
        when = destroy
        command = "./destroy_resource_group.sh"
        interpreter = ["bash", "-c"]
    }
}
```

## **Benefits**

- The ability to execute any command available in the Azure CLI using Terraform.

## **Downsides**

- The information about the resource we have created or modified is not present in the state file. The only data available in the state file is the one present on the ``triggers`` attribute.
- If an error is thrown during the script execution, your state file might end up in an inconsistent state. It depends on how you build the script.
- Destroying a resource requires a multi-step process.
- Everytime you want to provision a resource you have to write 2 scripts: one for provisioning the resource and another one for destroying it.
- You can't run an in-place update on a resource in a subsequent ``apply`` command, you can only create or destroy the resource.

# **Using the azurerm_resource_group_template_deployment resource + ARM Template**

The ``azurerm_resource_group_template_deployment`` is a resource from the official AzureRM Terraform provider and it allows us to manage a Resource Group Template Deployment using an ARM template.
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group_template_deployment

This approach is the equivalent of deploying an Azure ARM Template, but using Terraform to do it.

The next code snippet shows an example of how you could use the ``azurerm_resource_group_template_deployment`` to manage an ``Azure Network Watcher``.

```csharp
resource "azurerm_resource_group" "res_group" {
    name      = "rg-test"
    location  = "West US"
}

resource "azurerm_resource_group_template_deployment" "network_watcher" {
    name                  = "network-watcher-deployment"
    resource_group_name   = azurerm_resource_group.res_group.name
    deployment_mode       = "Incremental"
    template_content      = file("template.json")
    parameters_content  = <<PARAMETERS
    {
        "name": {
            "value": "network-wather-example"
        },
        "location": {
            "value": "${azurerm_resource_group.res_group.location}"
        }
    }
    PARAMETERS 
    depends_on = [
        azurerm_resource_group.res_group
    ]
}
```
And the ARM Template looks like this:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "String"
        },
        "location": {
            "type": "String"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Network/networkWatchers",
            "apiVersion": "2022-01-01",
            "name": "[parameters('name')]",
            "location": "[parameters('location')]",
            "properties": {}
        }
    ]
}
```

## **Benefits**

- The ability to deploy any resource available on the Azure ARM REST Api.

## **Downsides**

- When an ARM template execution fails in Terraform, Terraform doesn't record the fact that the deployment was physically created in the state file. Which means that the next time you try to deploy the Terraform ARM template it will be error out, from this point forward the only solution is to manually delete the Azure deployment.
- Terraform only understands changes made to the ARM template. If you modify a resource directly in Azure, Terraform is not capable to pick up those changes.
- It is not easy if you want to update an attribute of an existing resource. You'll have to describe the entire resource on your ARM template even if you only want to update a single property of the existing resource.


# **Using the AzAPI provider**

> **Important**: Using the ``AzApi`` provider is the best available solution when trying to create or update an Azure resource that is missing in the AzureRM provider.

The ``AzAPI`` provider is a very thin layer on top of the Azure ARM REST APIs (This API is the same one that is used when we deploy an ARM Template).   

The ``AzAPI`` provider contains only 3 resources:
- ``azapi_resource``:  Used for managing Azure resources.
- ``azapi_update_resource``: Used to add or modify properties on an existing resource. If you delete an ``azapi_update_resource`` block, no operation will be performed and these properties will stay unchanged. If you want to restore the modified properties, you must reapply the restored properties before deleting.
- ``azapi_resource_action``: Used to perform any resource action. It's recommended to use this resource to perform actions which change a resource state.

The next code snippet shows an example of how you could use the ``AzApi`` provider to manage an ``Azure DNS Resolver``.

```csharp
resource "azurerm_resource_group" "res_group" {
    name      = "rg-test"
    location  = "West US"
}

resource "azurerm_virtual_network" "vnet" {
    name                = "vnet-test"
    location            = azurerm_resource_group.res_group.location
    resource_group_name = azurerm_resource_group.res_group.name
    address_space       = ["10.18.0.0/16"]
}

resource "azapi_resource" "dns_resolver" {
    type      = "Microsoft.Network/dnsResolvers@2020-04-01-preview"
    name      = "resolver-test"
    parent_id = azurerm_resource_group.res_group.id
    location  = azurerm_resource_group.res_group.location

    body =  jsonencode({
        properties = {
            virtualNetwork = {
                id = azurerm_virtual_network.vnet.id
            }
        }
    })
    response_export_values = ["*"]
}
```

When creating or update a resource using the ``AzApi`` resource you'll need to specify the following attributes:
- ``type``: The value for this field is the resource-type and the api-version, following the convention: ``<resource-type>@<api-version>`` as an example ``Microsoft.Network/dnsResolvers@2020-04-01-preview``.

- ``parent_id`` Is the Id of the resource or item is deployed within, such as a resource group id.

- ``body``: Is the JSON object that contains the request body used to either create or update the Azure resource in question.   
  
## **Documentation**

This is the "go to" website when using the ``AzApi`` provider:
- https://learn.microsoft.com/es-es/azure/templates/?view=azurermps-6.0.0

You'll find an inventory of all the Azure resources available and how to create them using the ``AzApi`` provider.

## **Benefits**

- The ability to deploy any resource available on the Azure ARM REST Api.
- Full Terraform state file fidelity.
- It is really easy to update an attribute on an existing resource.
- There is no need to use any external file, such as an script or an ARM Template.
- It is the most "Terraform syntax friendly" from the 3 options we have discussed in this post.

## **Downsides**

- The fact that the ``body`` attribute is a ``jsonencode`` object might sometimes show you a false change when running the ``terraform plan`` command.


# **Examples**

During this post we have seen some really simple examples of how to create or update an Azure resource using the ``local-exec`` provisioner, the ``azurerm_resource_group_template_deployment`` resource or the ``AzApi`` provider. 

Before ending this post, let me show you a couple more complex ones.

# **Example 1: Create an Azure App Configuration**

I'm aware that you can create an ``Azure App Configuration`` using the ``azurerm_app_configuration`` resource available on the AzureRM provider, but this is just an example of how you can provision an Azure resource using Terraform without the AzureRM provider.

## **Using the null_resource, local-exec provisioner and the AZ CLI**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - A ``null_resource`` with a couple of ``local-exec`` provisioners to create an ``Azure App Configuration``. 
    - The first ``local-exec`` provisioner is triggered when the resource needs to be created and it invokes the ``create_app_config.sh`` script.
    - The second ``local-exec`` provisioner contains a ``when = destroy`` attribute, which mean that it will be triggered when the resource needs to be destroyed. This second provisioner will invoke the ``destroy_app_config.sh`` script.
  -  The ``Azure App Configuration`` creation/deletion is done inside the Shell scripts.
     -  The scripts uses the ``AZ CLI`` to create or delete the ``App Configuration``.

Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name = "rg-provisioning-demo"
    app_conf_name = "appconf-demo-dev"
    app_conf_sku = "Free"
    app_conf_enable_public_network = false
    app_conf_disable_local_auth = false
    app_conf_location = "westeurope"
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create App Configuration using an external script
resource "null_resource" "app_conf" {
    triggers = {
        app_conf_name = local.app_conf_name
        res_group_name = local.resource_group_name
        sku = local.app_conf_sku
        enable_public_network = local.app_conf_enable_public_network
        disable_local_auth = local.app_conf_disable_local_auth
        location = local.app_conf_location
    }
    provisioner "local-exec" {
        command = "./create_app_config.sh ${self.triggers.app_conf_name} ${self.triggers.res_group_name} ${self.triggers.sku} ${self.triggers.enable_public_network} ${self.triggers.disable_local_auth} ${self.triggers.location}" 
        interpreter = ["bash", "-c"]
    }

    provisioner "local-exec" {
        when = destroy
        command = "./destroy_app_config.sh ${self.triggers.app_conf_name} ${self.triggers.res_group_name} ${self.triggers.sku} ${self.triggers.enable_public_network} ${self.triggers.disable_local_auth}"
        interpreter = ["bash", "-c"]
    }

    depends_on = [
        azurerm_resource_group.rg_demo
    ]
}
```

The ``create_app_config.sh`` script executes the following steps:
- It checks if an App Configuration with the given name already exists.
- If it doesn't exists, it creates a new ``App Configuration`` using the ``az appconfig create`` command.

```bash
#!/bin/bash

APP_CONFIG_NAME="$1"
RES_GROUP_NAME="$2"
SKU="$3"
ENABLE_PUBLIC_NETWORK="$4"
DISABLE_LOCAL_AUTH="$5"
LOCATION="$6"

app_config_instance=$(az appconfig list | jq --arg name "$APP_CONFIG_NAME" -e '.[]|select(.name==$name).name')

if [ -z "$app_config_instance" ]; then
    echo "App Config doesn't exists. Creating a new one."
    az appconfig create --name $APP_CONFIG_NAME --resource-group $RES_GROUP_NAME --sku $SKU --enable-public-network $ENABLE_PUBLIC_NETWORK --disable-local-auth $DISABLE_LOCAL_AUTH --location $LOCATION
fi
```

The ``destroy_app_config.sh`` script executes the following steps:
- It checks if an App Configuration with the given name already exists.
- If it exists, it deletes the ``App Configuration`` using the ``az appconfig delete`` command.
- Sleeps during 30 seconds. 
 
With the ``local-exec`` provisioner we can't update a resource, the only option available is to delete it and afterwards recreate it with the new updated attributes.    

To update an ``App Configuration`` using this method, Terraform will delete it and then recreate it from scratch, if the resource creation happens right away after the deletion an error might appear saying that the ``App Configuration`` still exists, that's the reason why we're running the sleep command after deleting the resource, to let Azure enough time to know that the resource has been deleted.

```bash
#!/bin/bash

APP_CONFIG_NAME="$1"
RES_GROUP_NAME="$2"
SKU="$3"
ENABLE_PUBLIC_NETWORK="$4"
DISABLE_LOCAL_AUTH="$5"

app_config_instance=$(az appconfig list | jq --arg name "$APP_CONFIG_NAME" -e '.[]|select(.name==$name).name')

if [ -z "$app_config_instance" ]; then
    echo "App Config doesn't exist. No need to delete anything."
else
    echo "App Config found. Trying to destroy the resource."
    az appconfig delete --name $APP_CONFIG_NAME --resource-group $RES_GROUP_NAME --yes
    sleep 30s
fi
```

## **Using the azurerm_resource_group_template_deployment resource**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - An ``azurerm_resource_group_template_deployment``  to create the ``App Configuration``
    - This resource uses an ARM Template that can be found on the ``template.json`` file.
    - Parameters can be passed from the Terraform file to the ARM template using the ``parameters_content`` attribute.

Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name = "rg-provisioning-demo"
    app_conf_name = "appconf-demo-dev"
    app_conf_sku = "free"
    app_conf_public_network_access = "Enabled"
    app_conf_disable_local_auth = false
    app_conf_location = "westeurope"
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create App Configuration using the resource group arm template resource
resource "azurerm_resource_group_template_deployment" "appconf" {
    name                  = "app-conf-deploy"
    resource_group_name   = local.resource_group_name
    deployment_mode       = "Incremental"
    template_content      = file("template.json")
    parameters_content  = <<PARAMETERS
    {
        "name": {
            "value": "${local.app_conf_name}"
        },
        "location": {
            "value": "${local.app_conf_location}"
        },
        "sku": {
            "value": "${local.app_conf_sku}"
        },
        "public_network_access": {
            "value": "${local.app_conf_public_network_access}"
        },
        "disable_local_auth": {
            "value": "${local.app_conf_disable_local_auth}"
        }
    }
    PARAMETERS 
    depends_on = [
        azurerm_resource_group.rg_demo
    ]
}
```
And here's how the ARM template file looks like:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "String",
            "metadata": {
                "description": "Specifies the name of the App Configuration."
            }
        },
        "location": {
            "type": "String",
            "metadata": {
                "description": "Specifies the location of the App Configuration."
            }
        },
        "sku": {
            "type": "String",
            "metadata": {
                "description": "The SKU name of the App Configuration"
            }
        },
        "public_network_access": {
            "type": "String",
            "metadata": {
                "description": "The Public Network Access setting of the App Configuration"
            }
        },
        "disable_local_auth": {
            "type": "String",
            "metadata": {
                "description": "Whether local authentication methods is enabled."
            }
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.AppConfiguration/configurationStores",
            "apiVersion": "2022-05-01",
            "name": "[parameters('name')]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "[parameters('sku')]"
            },
            "properties": {
                "encryption": {},
                "publicNetworkAccess": "[parameters('public_network_access')]",
                "disableLocalAuth": "[parameters('disable_local_auth')]",
                "softDeleteRetentionInDays": 0,
                "enablePurgeProtection": false
            }
        }
    ]
}
```

## **Using AzApi provider**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - An ``azapi_resource``  to create the ``App Configuration``.

Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name = "rg-provisioning-demo"
    app_conf_name = "appconf-demo-dev"
    app_conf_sku = "free"
    app_conf_public_network_access = "Enabled"
    app_conf_disable_local_auth = false
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create App Configuration using the AzApi provider
resource "azapi_resource" "appconf" {
    type      = "Microsoft.AppConfiguration/configurationStores@2022-05-01"
    name      = local.app_conf_name
    parent_id = azurerm_resource_group.rg_demo.id
    location  = azurerm_resource_group.rg_demo.location
    body =  jsonencode({      
        sku = {
            name = local.app_conf_sku
        } 
        properties = {
            publicNetworkAccess = local.app_conf_public_network_access
            disableLocalAuth = local.app_conf_disable_local_auth
        }
    })
    response_export_values = ["*"]
}
```

# **Example 2: Update an existing Azure Storage Account to enable SFTP support**

This is an example that shows how you can update an existing resource without using the AzureRM Terraform provider.

To be more precise we're going to enable the [SFTP support for Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) on an existing Azure Storage Account.

> _Right now (11/09/2022) it is impossible to enable the [SFTP support for Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) using the AzureRM Terraform provider, so this is going to be a more realistic example than the previous one._

## **Using the null_resource, local-exec provisioner and the AZ CLI**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - An ``azurerm_storage_account`` resource to create a storage account.
  - An ``azurerm_storage_container`` resource to create a container within the storage account.
  - A ``null_resource`` with a couple of ``local-exec`` provisioners to enable the SFTP support and create a local user with permissions to access the SFTP. 
    - The first ``local-exec`` provisioner executes the ``enable_sftp_create_localuser.sh`` script that enables the SFTP support and creates an SFTP local user.
    - The second ``local-exec`` provisioner contains the ``when = destroy`` attribute. This provisioner is triggered when the resource needs to be destroyed and it invokes the ``disable_sftp_and_localuser.sh`` script that delete the SFTP local user.
  -  Enabling or disabling the SFTP support is done via Shell script.
     -  The Shell script uses the ``AZ CLI`` to enable or disable the SFTP support.


Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name = "rg-provisioning-demo"
    storage_account_name = "stsftprovdev"
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create Storage Account
resource "azurerm_storage_account" "sftp_storage_acct" {
    name                        = local.storage_account_name
    location                    = azurerm_resource_group.rg_demo.location
    resource_group_name         = azurerm_resource_group.rg_demo.name
    account_tier                = "Standard"
    account_replication_type    = "LRS"
    min_tls_version             = "TLS1_2"
    is_hns_enabled           = true
}

# Create container
resource "azurerm_storage_container" "azurerm_storage_container" {
  name                  = "container"
  storage_account_name  = azurerm_storage_account.sftp_storage_acct.name
}


## Enable SFTP and create SFTP local users using an external script
resource "null_resource" "sftp_enable" {
    triggers = {
        storage_account_name = local.storage_account_name
        res_group_name = local.resource_group_name
        sftp_user = "ftpuser"
    }
    provisioner "local-exec" {
        command = "./enable_sftp_create_localuser.sh ${self.triggers.storage_account_name} ${self.triggers.res_group_name} ${self.triggers.sftp_user}"
        interpreter = ["bash", "-c"]
    }

    provisioner "local-exec" {
        when = destroy
        command = "./disable_sftp_and_localuser.sh ${self.triggers.storage_account_name} ${self.triggers.res_group_name} ${self.triggers.sftp_user}" 
        interpreter = ["bash", "-c"]
    }

    depends_on = [
        azurerm_storage_container.azurerm_storage_container
    ]
}
```

The ``enable_sftp_create_localuser.sh`` script executes the following steps:
- Checks if the ``AllowSFTP`` feature is enabled in your Azure subscription, and if it is not enabled it throws and error.
- Checks if the ``storage-preview`` extension is installed on your machine. If it is not installed, it installs it.
- Enables SFTP support on the storage account.
- Creates the SFTP local user.
- Retrieves the user password.

```bash
#!/bin/bash
STORAGE_ACCT_NAME="$1"
RES_GROUP_NAME="$2"
SFTP_USER="$3"

state=$(az feature show --namespace Microsoft.Storage --name AllowSFTP |  jq '.properties.state')

if [ $state != '"Registered"' ]; then
    echo "Feature not registered. Registration is an asynchronous operation, it must be done manually."
    exit 1
fi

extension=$(az extension list | jq -e '.[]|select(.name=="storage-preview").name')
if [ -z "$extension" ]; then
    echo "Storage-preview extension missing. Installing it."
    az extension add -n storage-preview
fi

az storage account update -g $RES_GROUP_NAME -n $STORAGE_ACCT_NAME --enable-sftp true
az storage account local-user create --account-name $STORAGE_ACCT_NAME -g  $RES_GROUP_NAME -n $SFTP_USER --home-directory "container" --has-ssh-password true --has-ssh-key true --permission-scope permissions=rw service=blob resource-name=container
az storage account local-user regenerate-password --account-name $STORAGE_ACCT_NAME -g $RES_GROUP_NAME -n $SFTP_USER
```

The ``disable_sftp_and_localuser.sh`` script executes the following steps:
- Deletes the SFTP local user.


```bash
#!/bin/bash
STORAGE_ACCT_NAME="$1"
RES_GROUP_NAME="$2"
SFTP_USER="$3"

az storage account local-user delete --account-name $STORAGE_ACCT_NAME -g  $RES_GROUP_NAME -n $SFTP_USER
```

## **Using the azurerm_resource_group_template_deployment resource**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - An ``azurerm_storage_account`` resource to create a storage account.
  - An ``azurerm_storage_container`` resource to create a container within the storage account.
  - An ``azurerm_resource_group_template_deployment``  to enable the Storage Account SFTP support and to create an user for the SFTP.
    - The ``azurerm_resource_group_template_deployment``  resource uses an ARM Template that can be found on the ``template.json`` file.
    - The ARM template Storage Account only needs to set the ``isSftpEnabled`` attribute to ``true``, but the entire object must be described.
    - Any change we make on the Terraform ``azurerm_storage_account`` resource must also be changed on the ARM template, or it will get overriden. 
    - Parameters can be passed from the Terraform file to the ARM template using the ``parameters_content`` attribute.

Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name         = "rg-provisioning-demo"
    storage_account_name        = "stsftprovdev"
    storage_account_tier        = "Standard"
    storage_account_replication = "LRS"
    storage_account_min_tls     = "TLS1_2"
    storage_account_hns_enabled = true
    sftp_user                   = "ftpuser2"
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create Storage Account
resource "azurerm_storage_account" "sftp_storage_acct" {
    name                        = local.storage_account_name
    location                    = azurerm_resource_group.rg_demo.location
    resource_group_name         = azurerm_resource_group.rg_demo.name
    account_tier                = local.storage_account_tier
    account_replication_type    = local.storage_account_replication
    min_tls_version             = local.storage_account_min_tls
    is_hns_enabled              = local.storage_account_hns_enabled
}

# Create container
resource "azurerm_storage_container" "azurerm_storage_container" {
  name                  = "container"
  storage_account_name  = azurerm_storage_account.sftp_storage_acct.name
}

## Enable SFTP and add local users using the resource group arm template resource
resource "azurerm_resource_group_template_deployment" "sftp" {
    name                  = "sftp-deploy"
    resource_group_name   = local.resource_group_name
    deployment_mode       = "Incremental"
    template_content      = file("template.json")
    parameters_content  = <<PARAMETERS
    {
        "storage_account_name": {
            "value": "${local.storage_account_name}"
        },
        "storage_account_tier": {
            "value": "${local.storage_account_tier}"
        },
        "storage_account_replication": {
            "value": "${local.storage_account_replication}"
        },
        "storage_account_min_tls": {
            "value": "${local.storage_account_min_tls}"
        },
        "storage_account_hns_enabled": {
            "value": ${local.storage_account_hns_enabled}
        },
        "sftp_user": {
            "value": "${local.sftp_user}"
        }
    }
    PARAMETERS 
    depends_on = [
        azurerm_resource_group.rg_demo,
        azurerm_storage_account.sftp_storage_acct,
        azurerm_storage_container.azurerm_storage_container
    ]
}
```
And here's how the ARM template file looks like:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "storage_account_name": {
            "type": "String",
            "metadata": {
                "description": "Specifies the name of the Storage Account."
            }
        },
        "storage_account_tier": {
            "type": "String",
            "metadata": {
                "description": "Defines the Tier to use for this storage account."
            }
        },
        "storage_account_replication": {
            "type": "String",
            "metadata": {
                "description": "Defines the type of replication to use for this storage account."
            }
        },
        "storage_account_min_tls": {
            "type": "String",
            "metadata": {
                "description": "The minimum supported TLS version for the storage account."
            }
        },
        "storage_account_hns_enabled": {
            "type": "Bool",
            "metadata": {
                "description": "Is Hierarchical Namespace enabled?."
            }
        },
        "sftp_user": {
            "type": "String",
            "metadata": {
                "description": "The SFTP local username."
            }
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2022-05-01",
            "name": "[parameters('storage_account_name')]",
            "location": "westeurope",
            "sku": {
                "name": "[concat(parameters('storage_account_tier'), '_', parameters('storage_account_replication'))]",
                "tier": "[parameters('storage_account_tier')]"
            },
            "kind": "StorageV2",
            "identity": {
                "type": "None"
            },
            "properties": {
                "defaultToOAuthAuthentication": false,
                "publicNetworkAccess": "Enabled",
                "allowCrossTenantReplication": true,
                "isNfsV3Enabled": false,
                "isLocalUserEnabled": true,
                "isSftpEnabled": true,
                "minimumTlsVersion": "[parameters('storage_account_min_tls')]",
                "allowBlobPublicAccess": true,
                "allowSharedKeyAccess": true,
                "isHnsEnabled": "[parameters('storage_account_hns_enabled')]",
                "networkAcls": {
                    "bypass": "AzureServices",
                    "virtualNetworkRules": [],
                    "ipRules": [],
                    "defaultAction": "Allow"
                },
                "supportsHttpsTrafficOnly": true,
                "encryption": {
                    "services": {
                        "file": {
                            "keyType": "Account",
                            "enabled": true
                        },
                        "blob": {
                            "keyType": "Account",
                            "enabled": true
                        }
                    },
                    "keySource": "Microsoft.Storage"
                },
                "accessTier": "Hot"
            }
        },
        {
            "type": "Microsoft.Storage/storageAccounts/blobServices",
            "apiVersion": "2022-05-01",
            "name": "[concat(parameters('storage_account_name'), '/default')]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', parameters('storage_account_name'))]"
            ],
            "sku": {
                "name": "[concat(parameters('storage_account_tier'), '_', parameters('storage_account_replication'))]",
                "tier": "[parameters('storage_account_tier')]"
            },
            "properties": {
                "cors": {
                    "corsRules": []
                },
                "deleteRetentionPolicy": {
                    "allowPermanentDelete": false,
                    "enabled": false
                }
            }
        },
        {
            "type": "Microsoft.Storage/storageAccounts/localusers",
            "apiVersion": "2022-05-01",
            "name": "[concat(parameters('storage_account_name'), '/', parameters('sftp_user'))]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', parameters('storage_account_name'))]"
            ],
            "properties": {
                "hasSshPassword": true,
                "permissionScopes": [
                    {
                        "permissions": "rw",
                        "service": "blob",
                        "resourceName": "container"
                    }
                ],
                "homeDirectory": "container",
                "hasSharedKey": false,
                "hasSshKey": false
            }
        },
        {
            "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
            "apiVersion": "2022-05-01",
            "name": "[concat(parameters('storage_account_name'), '/default/container')]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts/blobServices', parameters('storage_account_name'), 'default')]",
                "[resourceId('Microsoft.Storage/storageAccounts', parameters('storage_account_name'))]"
            ],
            "properties": {
                "defaultEncryptionScope": "$account-encryption-key",
                "denyEncryptionScopeOverride": false,
                "publicAccess": "None"
            }
        }
    ]
}
```

## **Using AzApi provider**

- This example uses:
  - An  ``azurerm_resource_group`` to create a resource group.
  - An ``azurerm_storage_account`` resource to create a storage account.
  - An ``azurerm_storage_container`` resource to create a container within the storage account.
  - An ``azapi_update_resource``  to enable the SFTP support. 
    - The resource updated is the storage account that we have created previously using the ``azurerm_storage_account`` resource.
  - An ``azapi_resource`` to create the SFTP local user.
  - An ``azapi_resource_action`` to retrieve the SFTP local user password.

Here's how the Terraform file looks like:
```csharp
locals {
    resource_group_name = "rg-provisioning-demo"
}

## Create Resource Group
resource "azurerm_resource_group" "rg_demo" {
    name      = local.resource_group_name
    location  = "West Europe"
}

## Create Storage Account
resource "azurerm_storage_account" "sftp_storage_acct" {
    name                        = "stsftprovdev"
    location                    = azurerm_resource_group.rg_demo.location
    resource_group_name         = azurerm_resource_group.rg_demo.name
    account_tier                = "Standard"
    account_replication_type    = "LRS"
    min_tls_version             = "TLS1_2"
    is_hns_enabled              = true
}

# Create container
resource "azurerm_storage_container" "sftp_storage_acct_container" {
  name                  = "container"
  storage_account_name  = azurerm_storage_account.sftp_storage_acct.name
}

# Enable SFTP
resource "azapi_update_resource" "sftp_azpi_sftp" {
  type        = "Microsoft.Storage/storageAccounts@2021-09-01"
  resource_id = azurerm_storage_account.sftp_storage_acct.id

  body = jsonencode({
    properties = {
      isSftpEnabled = true
    }
  })

  depends_on = [
    azurerm_storage_account.sftp_storage_acct,
    azurerm_storage_container.sftp_storage_acct_container
  ]
  response_export_values = ["*"]
}

# Create local user
resource "azapi_resource" "sftp_local_user" {
  type        = "Microsoft.Storage/storageAccounts/localUsers@2021-09-01"
  parent_id = azurerm_storage_account.sftp_storage_acct.id
  name = "ftpuser"

  body = jsonencode({
    properties = {
      hasSshPassword = true,
      homeDirectory = "container"
      hasSharedKey = true,
      hasSshKey = false,
      permissionScopes = [{
        permissions = "rl",
        service = "blob",
        resourceName = "container"
      }]
    }
  })

  response_export_values = ["*"]

  depends_on = [
    azurerm_storage_account.sftp_storage_acct,
    azurerm_storage_container.sftp_storage_acct_container,
    azapi_update_resource.sftp_azpi_sftp
  ]
}

# Retrieve password
resource "azapi_resource_action" "generate_sftp_user_password" {
  type        = "Microsoft.Storage/storageAccounts/localUsers@2022-05-01"
  resource_id = azapi_resource.sftp_local_user.id
  action      = "regeneratePassword"
  body = jsonencode({
    username = azapi_resource.sftp_local_user.name
  })

  response_export_values = ["sshPassword"]

  depends_on = [
    azurerm_storage_account.sftp_storage_acct,
    azurerm_storage_container.sftp_storage_acct_container,
    azapi_update_resource.sftp_azpi_sftp,
    azapi_resource.sftp_local_user
  ]
}
```

# **Useful links**

- https://learn.microsoft.com/es-es/azure/templates/?view=azurermps-6.0.0
- https://registry.terraform.io/providers/Azure/azapi/1.0.0
- https://registry.terraform.io/providers/hashicorp/azurerm/3.30.0
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group_template_deployment