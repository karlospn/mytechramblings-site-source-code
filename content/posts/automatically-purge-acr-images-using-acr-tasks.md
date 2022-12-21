---
title: "How to automatically purge stale images from Azure Container Registry using ACR Tasks"
date: 2022-12-21T13:03:19+01:00
tags: ["azure", "containers", "docker", "terraform"]
description: "Keeping your container registry free of stale or unwanted images is a task that often gets overlooked when beginning working with containers in the enterprise. In this post, I want to show you how you can use ACR Tasks to automate this process when working with Azure Container Registry."
draft: false
---

Keeping your container registry free of stale or unwanted images is a task that often gets overlooked when beginning working with containers in the enterprise.    

If you have a container registry in any cloud provider, like ACR (Azure Container Registry) or ECR (AWS Elastic Container Registry), and you keep pushing and pushing new images and new versions of existing images without removing the stale ones, the first thing that you're going to notice is that your cloud bill keeps increasing because storing images is not free, but also managing the container registry will become more and more cumbersome as more and more images start to pile up. 

An universal solution when you want to remove old images of your container registry is to write some kind of script, which connects to the registry and removes the dated image versions. This script can be executed on demand, but in most cases you'll end up having a scheduled execution using a third party tool like: Azure Pipelines, Github Actions, Azure Functions, etc.

If you're using Azure Container Registry, a better alternative to purge container images is to use an **ACR Task**, and that's what I want to show you in this post.

# **What is ACR Tasks?**

ACR Tasks comprises a compute execution workflow within Azure Container Registry.    
It can perform some container related actions, like building your application image when the source code gets updated, keep your images up-to-date and a few others functionalities.   

A Task runs within ACR and uses an ephemeral virtual machine.

If you want to know more about ACR Tasks you can read it here:
- https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview

In this post we're going to create an ACR Task that uses the **ACR CLI** to purge old images.

# **Azure Container Registry CLI**

The ACR CLI allow us to interact with ACR, the currently supported commands include:

- **Tag**: to view all the tags of a repository and individually untag them.
- **Manifest**: to view the manifest of a given repository and delete them if necessary.
- **Purge**: to be able to delete all tags that are older than a certain date and that match a regex specified filter.

You can download the ACR CLI from its [Github page](https://github.com/Azure/acr-cli), and use it like any other CLI tool.

The ACR CLI is also integrated with the ACR Tasks, which means that you **can execute any ACR CLI command within an ACR Task**.   

Here's a basic example of how to use the ACR CLI to list all the available tags in a repository:
```shell
$ acr tag list -r acrtasksdemo --repository my-app
Listing tags for the "my-app" repository:
acrtasksdemo.azurecr.io/my-app:1.0.0
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:dev
acrtasksdemo.azurecr.io/my-app:prod
```

To execute an ACR Task, you can use the AZ CLI and the ``az acr run`` command. An ACR Task won't run on your local machine, **it will run on an ephemeral virtual machine in Azure**.

Here's the previous example I ran, but this time using an ACR Task to execute the ``acr tag list`` command on Azure. 
```shell
$ az acr run --cmd "acr tag list --repository my-app" --registry acrtasksdemo /dev/null
Queued a run with ID: cb2
Waiting for an agent...
2022/12/20 15:49:12 Alias support enabled for version >= 1.1.0, please see https://aka.ms/acr/tasks/task-aliases for more information.
2022/12/20 15:49:12 Creating Docker network: acb_default_network, driver: 'bridge'
2022/12/20 15:49:12 Successfully set up Docker network: acb_default_network
2022/12/20 15:49:12 Setting up Docker configuration...
2022/12/20 15:49:13 Successfully set up Docker configuration
2022/12/20 15:49:13 Logging in to registry: acrtasksdemo.azurecr.io
2022/12/20 15:49:14 Successfully logged into acrtasksdemo.azurecr.io
2022/12/20 15:49:14 Executing step ID: acb_step_0. Timeout(sec): 600, Working directory: '', Network: 'acb_default_network'
2022/12/20 15:49:14 Launching container with name: acb_step_0
Listing tags for the "my-app" repository:
acrtasksdemo.azurecr.io/my-app:1.0.0
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:dev
acrtasksdemo.azurecr.io/my-app:prod
2022/12/20 15:49:14 Successfully executed container: acb_step_0
2022/12/20 15:49:14 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.810056)
```

If we take a look in the Azure Portal, you'll see that an ACR Task has been executed inside our ACR.

![acr-task-execution](/img/acr-task-execution.png)

## **Running an ACR Task on schedule**

The previous example ran an ACR Task on-demand, but you can also run it on a regular basis using the ``az acr task create`` command and the ``schedule`` parameter.

The next code snippet shows the previous example I ran, but this time using an scheduled ACR Task to execute the ``acr tag list`` command every minute:

```shell
az acr task create --name "scheduledAcrTask" \
        --cmd "acr tag list --repository my-app" \
        --schedule "* * * * *" \
        --registry acrtasksdemo \
        --context /dev/null
```

If we go into the Azure Portal, you'll see that an scheduled ACR Task has been created.

![acr-scheduled-task](/img/acr-scheduled-task.png)

And if we take a look at the recent ACR Tasks run, we'll see how every minute the "scheduledAcrTask" gets executed.

![acr-scheduled-task-details](/img/acr-scheduled-task-details.png)

# **Using the purge command**

The ``purge`` command is one of the available commands on the ACR CLI and it allows us to delete tags and images in a single or in multiple repositories based on:
- The name of the repository.
- The name of the tags.
- The date when the image was pushed into the repository.

## **filter parameter**

This parameter allows us to specify which tags are going to be purged based on a regular expression. 

>**Important**: The ``filter`` parameter only purges tags, **it doesn't delete the images** per se. To delete the tags and the images in a single command you need to use the ``untagged`` parameter.

Let's see a few examples. 

I have 2 repositories on my ACR with the following tags.
```bash
$ acr tag list -r acrtasksdemo --repository my-app
Listing tags for the "my-app" repository:
acrtasksdemo.azurecr.io/my-app:1.0.0
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:dev
acrtasksdemo.azurecr.io/my-app:prod

$ acr tag list -r acrtasksdemo --repository another-app
Listing tags for the "another-app" repository:
acrtasksdemo.azurecr.io/another-app:1.0.0
acrtasksdemo.azurecr.io/another-app:2.0.0
acrtasksdemo.azurecr.io/another-app:dev
```

- Purge all images and tags from the "my-app" repository, except the ones that has the "dev" or "prod" tags.
```bash
$ acr purge --registry acrtasksdemo --filter 'my-app:^((?!prod|dev).)*$' --untagged --ago 0d --dry-run
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
This repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:1.0.0
Manifests for this repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app@sha256:4096d3149843a25b599b2b603a0cfc9983870511669092ad6df0f5dc2fc600b4
acrtasksdemo.azurecr.io/my-app@sha256:57a94fc99816c6aa225678b738ac40d85422e75dbb96115f1bb9b6ed77176166
acrtasksdemo.azurecr.io/my-app@sha256:aa81b44e52f1434cf8f4673c7db7a01601ed3d6c1b44539c868a4324e963fcdd

Number of tags to be deleted: 4
Number of manifests to be deleted: 3
```

- Purge all images and tags from all repositories, except those that has the "dev" or "prod" tags.
```bash
$ acr purge --registry acrtasksdemo --filter '.*:^((?!prod|dev).)*$' --untagged --ago 0d --dry-run
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
This repository would be deleted: another-app
acrtasksdemo.azurecr.io/another-app:2.0.0
acrtasksdemo.azurecr.io/another-app:1.0.0
Manifests for this repository would be deleted: another-app
acrtasksdemo.azurecr.io/another-app@sha256:579c1f1008733c26af5bfbd189eaa8209c344182b24ef839456e2f4d78923e58
This repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:1.0.0
Manifests for this repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app@sha256:4096d3149843a25b599b2b603a0cfc9983870511669092ad6df0f5dc2fc600b4
acrtasksdemo.azurecr.io/my-app@sha256:57a94fc99816c6aa225678b738ac40d85422e75dbb96115f1bb9b6ed77176166
acrtasksdemo.azurecr.io/my-app@sha256:aa81b44e52f1434cf8f4673c7db7a01601ed3d6c1b44539c868a4324e963fcdd

Number of tags to be deleted: 6
Number of manifests to be deleted: 4
```

- Purge all images from all repositories.
```bash
$ acr purge --registry acrtasksdemo --filter '.*:.*' --untagged --ago 0d --dry-run
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
This repository would be deleted: another-app
acrtasksdemo.azurecr.io/another-app:2.0.0
acrtasksdemo.azurecr.io/another-app:dev
acrtasksdemo.azurecr.io/another-app:1.0.0
Manifests for this repository would be deleted: another-app
acrtasksdemo.azurecr.io/another-app@sha256:579c1f1008733c26af5bfbd189eaa8209c344182b24ef839456e2f4d78923e58
acrtasksdemo.azurecr.io/another-app@sha256:5be2fee6cd04d4fadb78f301797ee118621ca7c8e8a251314f98a9f52380780e
This repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app:4.0.0
acrtasksdemo.azurecr.io/my-app:dev
acrtasksdemo.azurecr.io/my-app:3.0.0
acrtasksdemo.azurecr.io/my-app:prod
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:1.0.0
Manifests for this repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app@sha256:4096d3149843a25b599b2b603a0cfc9983870511669092ad6df0f5dc2fc600b4
acrtasksdemo.azurecr.io/my-app@sha256:57a94fc99816c6aa225678b738ac40d85422e75dbb96115f1bb9b6ed77176166
acrtasksdemo.azurecr.io/my-app@sha256:8aa1d6c1d510fadd745b9c007e5fd6ad6e3e15254a7476129b84255bd61d2389
acrtasksdemo.azurecr.io/my-app@sha256:aa81b44e52f1434cf8f4673c7db7a01601ed3d6c1b44539c868a4324e963fcdd
acrtasksdemo.azurecr.io/my-app@sha256:d6cd3f9f3d8aa2de3f176c047d7dcc016cdbef90e5e4c0f902b535b98573ffe5

Number of tags to be deleted: 9
Number of manifests to be deleted: 7
```

## **ago parameter**

This parameter allow us to purge all tags that are older than the specified value.

>**Important**: The ``ago`` parameter only purges tags, **it doesn't delete the images** per se. To delete the tags and the images in a single command you need to use the ``untagged`` parameter.

Here's an example:

- Purge all images and tags from all repositories that are older than 3 days.

``acr purge --registry acrtasksdemo --filter '.*:.*' --untagged --ago 3d --dry-run``

The ``ago`` parameter is always required when using the ``purge`` command, but if you don't care about the date you can set it to ``0d``.    
For example, the ``acr purge --registry acrtasksdemo --filter 'my-app:.*' --untagged --ago 0d`` command, purges all images and tags from the "my-app" repository without taking into account the date they were pushed into the registry.

## **keep parameter**

This parameter allow us to keep the latest number of tags.

>**Important**: The ``keep`` parameter only purges tags, **it doesn't delete the images** per se. To delete the tags and the images in a single command you need to use the ``untagged`` parameter.

The following example will keep the latest 3 tags and images of "my-app" repository, and remove the remaining ones.
```bash
acr purge --registry acrtasksdemo --filter 'my-app:.*' --keep 3 --untagged --ago 0d --dry-run
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
This repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app:prod
acrtasksdemo.azurecr.io/my-app:2.0.0
acrtasksdemo.azurecr.io/my-app:1.0.0
Manifests for this repository would be deleted: my-app
acrtasksdemo.azurecr.io/my-app@sha256:57a94fc99816c6aa225678b738ac40d85422e75dbb96115f1bb9b6ed77176166
acrtasksdemo.azurecr.io/my-app@sha256:8aa1d6c1d510fadd745b9c007e5fd6ad6e3e15254a7476129b84255bd61d2389
acrtasksdemo.azurecr.io/my-app@sha256:aa81b44e52f1434cf8f4673c7db7a01601ed3d6c1b44539c868a4324e963fcdd

Number of tags to be deleted: 3
Number of manifests to be deleted: 3
```

## **dry-run parameter**

To know which tags and images would be deleted the ``dry-run`` flag can be set, nothing will be deleted and the output would be the same as if the purge command was executed normally.


# **Create a scheduled ACR Task that purges stale images from an ACR**

Now that we know how the ``purge`` command works, it is time to create an scheduled ACR Task that purges the stale images once every week.

Obviously, every one will have different requirements when purging images from an ACR repository, mainly because there are dozens of viable tag strategies you can follow when working with containers.   
But, for this post I'm going to create the following ACR Task:
- It will be an scheduled ACR Task that runs once every week.
- It will purge all the images that are older than 30 days from all the existing repositories, except the images with the "dev" or "prod" tag assigned.

Let's build the command we need:
- To purge all tags from all the existing repositories and keep only the "dev" and "prod" tag, we need to specify the following value for the ``filter`` parameter ``.*:^((?!prod|dev).)*$``
  - With the ``.*`` regular expression we're targeting all the repositories available in the ACR.
  - With the ``^((?!prod|dev).)*$`` regular expression we're targeting all the tags except "prod" or "dev".
- To purge all tags older than 30days, we need to set the ``ago`` parameter to: ``--ago 30d``
- The ``filter`` and ``ago`` parameter doesn't delete the manifest per se, it only purges the tags. To remove the images we have to use the ``untagged`` attribute.
- To run the ACR Task once every week we're going to use the ``schedule`` attribute alongside a CRON value. For example, this one: ``55 18 * * Tue``, which means that the ACR Task will run every Tuesday at 18:55 UTC Time.

Here's how the end result looks like:
```bash
  az acr task create \
      --name "scheduledAcrPurgeTask" \
      --cmd "acr purge --filter '.*:^((?!prod|dev).)*$' --untagged --ago 30d" \
      --schedule "55 18 * * Tue" \
      --registry acrtasksdemo \
      --context /dev/null
```      

If we take a look at the Azure portal after the ACR Task ran for the first time, we will see how it has deleted every image and tag older than 30 days except the ones with the "prod" or "dev" tags associated.

![acr-purge-scheduled-dev-prod-task](/img/acr-purge-scheduled-dev-prod-task.png)

If we take a closer look at one of the repositories, the only remaining images are the ones with the "prod" or "dev" tags on them.

![acr-purge-scheduled-dev-prod-task-repository](/img/acr-purge-scheduled-dev-prod-task-repository.png)


# **Terraform implementation**

As a bonus section, let me show you how you can create the same scheduled ACR Task using Terraform and the AzureRM provider.

```javascript
resource "azurerm_container_registry_task" "acr_purge_task" {
  name                  = "scheduledAcrPurgeTask"
  container_registry_id = azurerm_container_registry.acr.id
  platform {
    os           = "Linux"
    architecture = "amd64" 
  }
  encoded_step {
    task_content = <<EOF
version: v1.1.0
steps: 
  - cmd: acr purge --filter '.*:^((?!prod|dev).)*$' --untagged --ago 30d
    disableWorkingDirectoryOverride: true
    timeout: 3600
EOF
  }
  agent_setting {
    cpu = 2
  }
  timer_trigger {
    name     = "t1"
    schedule = "55 18 * * Tue"
    enabled  = true
  }
}
```