---
title: "How to deploy an Azure resource using Terraform when it is not available in the AzureRM official provider"
date: 2022-11-03T23:10:40+01:00
tags: ["azure", "devops", "iac", "terraform", "cloud"]
description: "TBD"
draft: true
---

Every one who has worked long enough with Terraform in Azure has been in the position of wanting to deploy a resource that's not available on the official [Azure Terraform provider](https://registry.terraform.io/providers/hashicorp/azurerm).   

The same situation also happens when trying to enable a feature of an existing resource and that feature is missing from the AzureRM Terraform provider.

A solution to those problems might be to switch to Bicep or Azure ARM Templates, but if all my cloud infrastructure is written as code using Terraform, why should I switch to another tool?What can I do to keep using Terraform?

Well, that's the point of this post, **to review which options we have available when we want to create or update a service on Azure and it is not available on the Azure Terraform provider.**

Before we dive further into this post, let me remind you that if you can avoid using any of the options we're going to tackle in this post then do so.    
Doing what we're going to discuss here should be a last resort and it makes sense only **when the AzureRM provider doesn't have a native implementation for the service we want to create or update**.

# **Available options for creating or updating an Azure resource when it is not available in the AzureRM provider**

Nowadays there are **3 options available**:
- Using the  ``null_resource`` resource alongside with the ``local-exec`` provisioner that executes a script using the ``Azure CLI`` or ``Azure Az Powershell Module``.
- Using the ``azurerm_resource_group_template_deployment`` resource from the AzureRM provider alongside an ``ARM Template``.
- Using the ``AzAPI`` provider.

# **Using the null_resource + local-exec provisioner + Azure CLI or Azure Az Powershell Module**

> **Important**: Use this approach as a **last resort**. The other options discussed in this post are far better alternatives.

The  ``null_resource``  is a Terraform resource that doesn't do anything.   
If you need to run provisioners that aren't directly associated with a specific resource, you can associate them with a ``null_resource``.  

The ``local-exec`` provisioner is used to invoke a local executable on the machine running Terraform.

Pairing the ``null_resource``  with the ``local-exec`` provisioner allows us to define a Terraform resource that can run a local executable on our local machine, such as an external script that uses the Azure CLI to create a new resource on Azure.

The next code snippet shows an example of how you could execute a script that creates an Azure Resource Group using  the ``null_resource`` and the ``local-exec`` provisioner.

```yaml
resource "null_resource" "az_res_group" {
  triggers = {
    location = "westus
  }
  provisioner "local-exec" {
    command = "./create_resource_group.sh ${self.triggers.location}"
    interpreter = ["/bin/bash", "-c"]
  }
}
```
And the ``create_resource_group.sh`` script will look like:

```shell
#!/bin/bash
LOCATION="$1"
az group create -l $LOCATION -n MyResourceGroup
```

The ``triggers`` argument allows specifying an arbitrary set of values that, when changed, will cause the resource to be replaced.

## **How to destroy a resource using the local-exec provisioner**

By default, the ``local-exec`` provisioner will run only when the resource they are defined within is created. It only runs during resource creation, not during updating or any other lifecycle.

If we want that our resource gets deleted using the ``local-exec`` provisioner we need to use the ``when`` argument.   
If ``when = destroy`` is specified, the provisioner will run when the resource is destroyed.

The next code snippet shows an example of how you could execute a script that creates and deletes an Azure Resource Group using  the ``null_resource`` and the ``local-exec`` provisioner.

```yaml
resource "null_resource" "az_res_group" {
  triggers = {
    location = "westus
  }

  provisioner "local-exec" {
    command = "./create_resource_group.sh ${self.triggers.location}"
    interpreter = ["/bin/bash", "-c"]
  }

  provisioner "local-exec" {
    when = destroy
    command = "./destroy_resource_group.sh ${self.triggers.location}"
    interpreter = ["/bin/bash", "-c"]
  }
}
```
But there is one big problem with this approach.

When we remove a resource from the Terraform file and execute the ``apply`` command normally the resource gets deleted from the Terraform state file and from the provider. That's the common behaviour of Terraform, but it doesn't apply to the ``local-exec`` provisioner.

If a  ``null_resource`` block with a ``local-exec``  gets removed entirely from the Terraform file the resource **won't be destroyed** at all. To work around this, a multi-step process needs to be used to safely remove a resource:
   - Update the resource configuration to include count = 0.
   - Apply the configuration to destroy any existing instances of the resource, including running the destroy provisioner.
   - Remove the resource block entirely from configuration, along with its provisioner blocks.
   - Apply again, at which point no further action should be taken since the resources were already destroyed.


## **Downsides**

Using this approach has a few downsides:
- The information about the resource we have created or modified is not present in the state file. The only data available in the state file is the one from the ``triggers`` attributes.
- Depending on how you build the script and if it errors out when it gets executed, it might happen that your state ends up in a inconsistent state.
- To destroy a resource is not an easy feat.
- If you want to be consistent you'll have to write not one but 2 scripts: one for provisioning the resource and one for destroying the resource.
- Using this approach you can create or destroy a resource, but you can't update a resource.


## **Benefits**

- The ability to execute any command available in the Azure CLI using Terraform.


# **Using the azurerm_resource_group_template_deployment resource + ARM Template**

The ``azurerm_resource_group_template_deployment`` is a resource from the official Azure Terraform provider and it allows us to manage a Resource Group Template Deployment using an ARM template.
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group_template_deployment

This approach is the equivalent of deploying an Azure ARM Template, but using Terraform to do it.

The next code snippet shows an example of how you could use the ``azurerm_resource_group_template_deployment`` to manage an Azure Resource Group.

```yaml
resource "azurerm_resource_group_template_deployment" "az_res_group" {
    name                  = "res-group-deploy"
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

## **Downsides**

- When an ARM template execution fails in Terraform, Terraform doesn't record the fact that the deployment was physically created in the state file. Which means that the next time you try to deploy the Terraform ARM template it will be error out, from this point forward the only solution is to manually delete the Azure deployment.
- Terraform only understands changes made to the ARM template. If you modify a resource directly in Azure, Terraform is not capable to pick up those changes.
- And you need to write ARM templates...

## **Benefits**

- The ability to deploy any resource available via Azure ARM REST Api.


# **Using the AzAPI provider**

The ``AzAPI`` provider is a very thin layer on top of the Azure ARM REST APIs (This API is the same one that is used when we deploy an ARM Template).   


# **Examples**

During this post we have seen some really simple examples of how to create or update an Azure resource using the ``local-exec`` provisioner, the ``azurerm_resource_group_template_deployment`` resource or the ``AzApi`` provider. 

Before ending this post, let me show you a more complex ones.


