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

# **Purging stale images using the ACR CLI**

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