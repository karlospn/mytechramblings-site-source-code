---
title: "Automatically purge container images hosted on ACR using ACR Tasks"
date: 2022-12-18T22:55:19+01:00
tags: ["azure", "containers", "docker", "terraform"]
description: "Keeping your container registry free of stale or unwanted container images is a task often gets overlooked when a company starts working with containers. The best way to purge container images when using ACR (Azure Container Registry) is to use ACR Tasks, and that's what I want to show you in this post."
draft: true
---

Keeping your container registry free of stale or unwanted container images is a task often gets overlooked when a company starts working with containers. If you're using a container registry in a cloud environment, like ACR (Azure Container Registry) or ECR (AWS Elastic Container Registry), and you keep pushing new versions of a container image into the container registry and you never purge the stale image version your cloud bill is going to keep increasing.

A universal solution when you want to remove old images of your container registry is to write some kind of a script that connects to the registry and removes the stale container images. This script can be executed on demand or it can have a scheduled execution using a third party tool like: Azure Pipelines, Github Actions, Azure Function, etc.

But a preferred way to purge container images when using ACR (Azure Container Registry) is to use **ACR Tasks**, and that's what I want to show you in this post.

# **What is ACR Tasks?**

ACR Tasks is a suite of features within Azure Container Registry. 

In this post we're going to use it to automatically purge old container images, but it has quite a few more functionalities, for example ACR Tasks can be used to:
- Build your application container image when the source code gets updated. 
- Automate OS and framework patching for your container images when the source code gets updated. 

If you want to know more about ACR Tasks you can read it here:
- https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview


# **Azure Container Registry CLI**

The ACR CLI allow us to interact with ACR, the currently supported commands include

- **Tag**: to view all the tags of a repository and individually untag them.
- **Manifest**: to view the manifest of a given repository and delete them if necessary.
- **Purge**: to be able to delete all tags that are older than a certain date and that match a regex specified filter.

You can download the ACR CLI from its [Github page](https://github.com/Azure/acr-cli), and use it like any other CLI  tool.   
Here's a simple example where I list all the available tags in an ACR repository:

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

The ACR CLI is also integrated with the ACR Tasks, which means that you **can delegate the execution of any ACR CLI command to an ACR Task**.

To execute an ACR Task, you can use the AZ CLI using the ``az acr run`` command. This task won't run on your local machine, **it will run inside ACR itself using ACR own computing resource**.

Here's the previous example I ran, but using an ACR Task to execute the ``acr tag list`` comand. 
```shell
$ az acr run --cmd "acr tag list -r acrtasksdemo --repository my-app" --registry acrtasksdemo /dev/null
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

If we go into the Azure Portal, you can see that an ACR Task has been executed inside our ACR.

![acr-task-execution](/img/acr-task-execution.png)

## **Running an ACR Task on schedule**

The previous example ran an ACR Task on-demand, but you can also run it on a regular basic using the ``az acr task create`` command with the ``schedule`` attribute.

Here's an example of how to create an ACR Task that runs every minute and lists all the available tags of "my-app" repository:   
``az acr task create --name "scheduledAcrTask" --cmd "acr tag list -r acrtasksdemo --repository my-app" --schedule "* * * * *" --registry acrtasksdemo --context /dev/null``

If we go into the Azure Portal, you can see that an scheduled ACR Task has been created:

![acr-scheduled-task](/img/acr-scheduled-task.png)

And if we take a look at the recent ACR Tasks run, we can see how every minute the "scheduledAcrTask" gets executed:

![acr-scheduled-task-details](/img/acr-scheduled-task-details.png)

# **Using the purge command**

The ``purge`` command is one of the commands available on the ACR CLI, it allow us to delete images in one or multiple repositories based on:
- A regexp filter.
- When the image was pushed.
- If the image has a tag associated or not.

## **dry-run parameter**

To know which tags and manifests would be deleted the ``dry-run`` flag can be set, nothing will be deleted and the output would be the same as if the purge command was executed normally.

The following example will purge the image tag as "2.0.0 from the "another-app" repository, but because the ``dry-run`` parameter is specified nothing will be deleted.
```bash
$ acr purge --registry acrtasksdemo --filter 'another-app:2.0.0' --ago 0d --dry-run
DRY RUN: The following output shows what WOULD be deleted if the purge command was executed. Nothing is deleted.
This repository would be deleted: another-app
acrtasksdemo.azurecr.io/another-app:2.0.0

Number of tags to be deleted: 1
Number of manifests to be deleted: 0
```

## **filter parameter**

This parameter allows us to specify which tags are going to be purged based on a regular expression. 

>**Important**: The ``filter`` parameter only purges tags, it doesn't delete the image per se, the manifest is still available in the repository. To delete the image completely you need to use it alongside the ``untagged`` parameter.

Let's see a few examples. 

I have those 2 repositories with the following tags.
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

- To purge all images and tags from the "my-app" repository except those that contains the "dev" or "prod" tags.
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

- To purge all images and tags from all repositories, except those that contains the "dev" or "prod" tags.
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

- To purge all images from all repositories.
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

>**Important**: The ``ago`` parameter only purges tags, it doesn't delete the image per se, the manifest is still available in the repository. To delete the image completely you need to use it alongside the ``untagged`` parameter.

Here's some examples:

- To purge all images from all repositories that are older than 3 days.

``acr purge --registry acrtasksdemo --filter '.*:.*' --untagged --ago 3d --dry-run``

The ``ago`` parameter is a required one, but if you don't care about it you can set it to ``0d``.    
For example, with this command: ``acr purge --registry acrtasksdemo --filter '.*:.*' --ago 0d --dry-run`` we're telling the ACR CLI to purge all images and all tags from all repositories and it doesn't matter when the images were pushed into the registry.

## **keep parameter**

This parameter allow us to keep the latest number of tags.

>**Important**: The ``keep`` parameter only purges tags, it doesn't delete the image per se, the manifest is still available in the repository. To delete the image completely you need to use it alongside the ``untagged`` parameter.

The following example will keep the 3 latest tags and images of "my-app" repository, and purge the remaining ones.
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

# **Create a scheduled ACR Task that purges stale images from ACR**

Now that we know how the ``purge`` command works, it is time to create an scheduled ACR Task that purges the stale images once every week.

Obviously, every one will have different requirements when purging images from an ACR repository, mainly because there are quite a few common tag strategies you can follow when working with containers.

For this post I'm going to create the following ACR Task:
- It will be an scheduled ACR Task that runs once every week
- It will purge all the images that are older than 30 days from all the existing repositories, except the images with the "dev" or "prod" tag assigned.

``az acr task create --name "scheduledAcrPurgeTask" --cmd "acr purge --registry acrtasksdemo --filter '.*:^((?!prod|dev).)*$' --untagged --ago 30d" --schedule "0 1 * * Mon" --registry acrtasksdemo --context /dev/null``


# **Terraform implementation**

```yaml
## ACR Task purge
resource "azurerm_container_registry_task" "acr_purge_task" {
  name                  = "acrPurgeTask"
  container_registry_id = azurerm_container_registry.acr.id
  platform {
    os           = "Linux"
    architecture = "amd64" 
  }
  encoded_step {
    task_content = <<EOF
version: v1.1.0
steps: 
  - cmd: acr purge  --filter '.*:.*' --ago 0d --keep 5 --untagged
    disableWorkingDirectoryOverride: true
    timeout: 3600
EOF
  }
  agent_setting {
    cpu = 2
  }
  timer_trigger {
    name     = "t1"
    schedule = "0 1 * * Mon"
    enabled  = true
  }
}
```