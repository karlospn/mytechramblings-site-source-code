---
title: "How to deploy an Azure resource using Terraform when it is not available in the AzureRM official provider"
date: 2022-11-03T23:10:40+01:00
tags: ["azure", "devops", "iac", "terraform", "cloud"]
description: "TBD"
draft: true
---

Every one who has worked long enough with Terraform in Azure has been in the position of wanting to deploy a resource that's not available on the official [Azure Terraform provider](https://registry.terraform.io/providers/hashicorp/azurerm).   

The same situation also happens when trying to enable a feature of an existing resource and that feature is missing from the Azure Terraform provider.

A solution to those problems might be to switch to Bicep or Azure ARM Templates, but if all my cloud infrastructure is written as code using Terraform, why should I switch to another tool?What can I do to keep using Terraform?

Well, that's the point of this post, **to review which options we have available when we want to create or update an Azure service and it is not available on the AzureRM Terraform provider.**

Before we dive further into this post, let me remind you that if you can avoid using any of the options we're going to tackle in this post then do so.    
Doing what we're going to discuss here should be a last resort and it makes sense only **when the AzureRM provider doesn't have an implementation for the service we want to create or update**.

# **Available options for creating or updating an Azure resource when it is not available in the AzureRM provider**

Nowadays there are **3 options available**:
- Using the ``null_resource`` resource alongside with ``local-exec`` provisioner, and building a script that uses the ``Azure CLI`` or ``Azure Az Powershell Module``.
- Using the ``azurerm_resource_group_template_deployment`` resource from the AzureRM provider alongside an ``ARM Template``.
- Using the ``AzAPI`` provider.

# **Using the null_resource resource + local-exec provisioner + Azure CLI or Azure Az Powershell Module**

> **Important**: Use this approach as a **last resort**. The other options discussed in this post are far better alternatives.

If you need to run provisioners that aren't directly associated with a specific resource, you can associate them with a ``null_resource``.   
Instances of ``null_resource`` are treated like normal resources, but they don't do anything. 

The ``local-exec`` provisioner invokes a local executable on the machine running Terraform.

Pairing the ``null_resource`` resource with ``local-exec`` provisioner allows us to define a Terraform resource that can execute an external script.

The next code snippet shows an example of how to execute a shell script using the ``null_resource`` resource with a ``local-exec`` provisioner.

```yaml
resource "null_resource" "health_check" {
 provisioner "local-exec" {
    command = "/bin/bash healthcheck.sh"
  }
}
```



# **Using the azurerm_resource_group_template_deployment resource + ARM Template**

The ``azurerm_resource_group_template_deployment`` is a resource from the official Azure Terraform provider and it allows us to manages a Resource Group Template Deployment using an ARM template.

This resource works a wildcard because it allows us to deploy an Azure ARM Template, which means that we can deploy or update any resource available on Azure.

When an ARM template execution fails in Terraform, Terraform doesn't record the fact that the deployment was physically created in the state file. Consequently, to rerun the terraform ARM template after corrections, you either need to manually delete the Azure deployment, to do a Terraform import for that deployment to re-execute the Terraform configuration. 


# **Using the AzAPI provider**

The ``AzAPI`` provider is a very thin layer on top of the Azure ARM REST APIs (This API is the one that also uses to dpeloy ARM Templates).   


# **Examples**

## **Create a new resource**

## **Update an existing resource**