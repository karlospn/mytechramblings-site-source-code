---
title: "How to push a container image to Azure Container Registry (ACR) using Terraform"
date: 2024-01-15T11:50:00+01:00
description: "In this brief post, I want to show 3 options you can use to push an application into an Azure Container Registry (ACR) using Terraform."
tags: ["azure", "terraform", "iac"]
draft: false
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/how-to-push-a-container-image-into-acr-using-terraform).


In a more DevOps-oriented environment, where the application is first deployed into a container registry and then the necessary infrastructure is deployed to run that app, you might not face the issue I'm trying to solve in this post.

However, in a situation where the development and operations teams are separated, this becomes a more frequent problem. But what's the problem I'm trying to address here?

In a situation where the dev and ops teams are separated, the person creating the infrastructure needs to deploy the service first, and then the person responsible for deploying the app deploys it within the provided infrastructure.

This creates a chicken-and-egg problem. The person creating the infrastructure needs to reference some kind of application when creating the infrastructure, but the actual application doesn't exist yet because the person responsible for deploying the app doesn't have the infrastructure in place.

Let me make a quick break here: The specific service is not the main concern here, as this chicken-and-egg problem occurs in many Cloud PaaS services, such as Azure Web Apps, Azure Container Apps, Azure Container Instance, AWS ECS EC2, and AWS Fargate, to name a few.

A common solution to this problem is to reference a "dummy" application when deploying the infrastructure.   

You wouldn't want to create the cloud service referencing a public random Docker Hub image, because when the development team deploys the app into its private container registry, the ops team will need to modify the service so that it now points to the private container registry instead of the public one.  

This is why a "dummy" application is created that mimics the real-life application you will eventually replace in the service. This "dummy" application will be deployed using the same ports as the real application, the same health-checking endpoints, and so on. So when the development team swaps the "dummy" app for the real one, everything works flawlessly.

In this brief post, I want to demonstrate how you can use Terraform to push a "dummy" application into an Azure Container Registry (ACR) using Terraform.   

You could use a dozen different tools to accomplish this task, but if all your infrastructure is being deployed using Terraform, it makes sense to also create the "dummy" application using Terraform, so you won't need to mix tools.

I'm going to show you 3 options for pushing a container image to Azure Container Registry using Terraform, so let's get down to it.

# **Using the Terraform Docker provider**

> **If you want to review the source code for this scenario, you can click [here](https://github.com/karlospn/how-to-push-a-container-image-into-acr-using-terraform/tree/main/src/using_docker_provider).**

This scenario utilizes the **Terraform Docker provider** to built the "dummy" application image and push it to the Azure Container Registry (ACR). 

This provider uses the ACR admin user and password to log in to the ACR, so when you create the ACR, you must enable this option.

The following code snippet shows how to set up the Docker provider using the ACR address, ACR admin user, and ACR admin password.

```yml
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">=3.86.0"
    }
    docker = {
      source  = "kreuzwerker/docker"
      version = "3.0.2"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "docker" {
  registry_auth {
    address  = data.azurerm_container_registry.acr.login_server
    username = data.azurerm_container_registry.acr.admin_username
    password = data.azurerm_container_registry.acr.admin_password
  }
}
```

Let me make an important remark here, **this is NOT an official Terraform provider built by Docker**. There isn't an official one in existence.    
This is a provider built by some external company, it has a few quirks, but it is well-built overall.

Here's the link to the provider if you want to take a look:
- https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs

Also, it is crucial to note that to use this provider, **you'll need to have Docker installed and running on your local machine or wherever you choose to run it.**


Once you have set up the provider, we will be using the ``docker_image`` resource to create the "dummy" app and the ``docker_registry_image`` resource to push it to ACR.    
The following code snippet illustrates exactly this.
```yaml
# Set container image name
locals {
  image_name = "init-app-docker-provider:latest"
}

# Create container image
resource "docker_image" "init_app" {
  name = "${data.azurerm_container_registry.acr.login_server}/${local.image_name}"
  keep_locally = false
  
  build {
    no_cache = true
    context = "${path.cwd}/init-app"
  }

  triggers = {
    dir_sha1 = sha1(join("", [for f in fileset(path.cwd, "init-app/*") : filesha1(f)]))
  }
}

# Push container image to ACR
resource "docker_registry_image" "push_image_to_acr" {
  name          = docker_image.init_app.name
  keep_remotely = false
  
  triggers = {
    dir_sha1 = sha1(join("", [for f in fileset(path.cwd, "init-app/*") : filesha1(f)]))
  } 
}
```

As you can see from the above code snippet, it is pretty straightforward. The most interesting thing to note is the ``dir_sha1`` trigger.

In Terraform 0.12 and later, one can use a ``for`` expression combined with a ``fileset`` function and one of the hashing functions to calculate a combined checksum for files in a directory.

The ``dir_sha1`` trigger above will calculate a SHA1 checksum for each file in the "dummy" application directory, join the checksums in a string, and finally, calculate a checksum for the resulting string.

Basically, the ``dir_sha1`` trigger is used to force the creation and pushing of a new container image in case we need to modify something in the "dummy" app.

# **Using the Terraform null_resource resource alongside with the AZ CLI**

> **If you want to review the source code for this scenario, you can click [here](https://github.com/karlospn/how-to-push-a-container-image-into-acr-using-terraform/tree/main/src/using_null_resource_with_az_cli).**

This scenario utilizes the Terraform ``null_resource`` resource to execute a series of AZ CLI commands to built the application image and push it to the Azure Container Registry (ACR).

The AZ CLI command executed in this scenario is the ``az acr build`` command.

```text
az acr build --registry
             [--agent-pool]
             [--auth-mode {Default, None}]
             [--build-arg]
             [--file]
             [--image]
             [--log-template]
             [--no-format]
             [--no-logs]
             [--no-push]
             [--no-wait]
             [--platform]
             [--resource-group]
             [--secret-build-arg]
             [--target]
             [--timeout]
             [<SOURCE_LOCATION>]
```

The following code snippet shows how we're using the Terraform ``null_resource`` resource to invoke a shell script that runs the ``az acr build`` command.
```yaml
# Set container image name
locals {
  image_name = "init-app-null-resource-with-az-cli"
  image_tag = "latest"
}

# Create docker image
resource "null_resource" "docker_image" {
    triggers = {
        image_name = local.image_name
        image_tag = local.image_tag
        registry_name = data.azurerm_container_registry.acr.name
        dockerfile_path = "${path.cwd}/init-app/Dockerfile"
        dockerfile_context = "${path.cwd}/init-app"
        dir_sha1 = sha1(join("", [for f in fileset(path.cwd, "init-app/*") : filesha1(f)]))
    }
    provisioner "local-exec" {
        command = "./scripts/docker_build_and_push_to_acr.sh ${self.triggers.image_name} ${self.triggers.image_tag} ${self.triggers.registry_name} ${self.triggers.dockerfile_path} ${self.triggers.dockerfile_context}" 
        interpreter = ["bash", "-c"]
    }
}
```
And this is how simple the Shell script looks like.
```shell
#!/bin/bash
IMAGE_NAME="$1"
IMAGE_TAG="$2"
REGISTRY_NAME="$3"
DOCKERFILE_PATH="$4"
DOCKERFILE_CONTEXT="$5"

az acr build -t $IMAGE_NAME:$IMAGE_TAG -r $REGISTRY_NAME -f $DOCKERFILE_PATH $DOCKERFILE_CONTEXT
```

As you can see from the above Terraform code snippet, we're using the ``sha1``as a trigger again to force the creation and pushing of a new container image in case we need to modify something in the "dummy" app.

- **However, what's the advantage of this scenario over the previous one?**

The ``null_resource`` is typically more complex to use and is often considered a last resort in Terraform workflows.

In this case, the benefit of using it alongside the ``az acr build command`` is that it allows us to execute it on a machine that doesn't have Docker installed. This is because the ``az acr build`` command simply queues a build job within the ACR, eliminating the need to have Docker installed on your local machine.

![terraform-push-app-az-acr-build-command](/img/terraform-push-app-az-acr-build-command.png)

Another advantage of using this scenario is that you don't need to enable the admin user and password on the ACR, addressing a common security concern.

To run the ``az acr build`` command, you need at least to have the ``Contributor`` role in Azure.


# **Using the Terraform null_resource resource and Docker commands**

> **If you want to review the source code for this scenario, you can click [here](https://github.com/karlospn/how-to-push-a-container-image-into-acr-using-terraform/tree/main/src/using_null_resource_with_docker_commands).**


This scenario employs the Terraform ``null_resource`` resource to execute a sequence of Docker commands for building the application image and pushing it to the Azure Container Registry (ACR).

This scenario, as the first one, uses the ACR admin user and password to log in to the ACR, so when you create the ACR, you must enable this option.

The Docker commands that will be executed in this scenario are the following ones:
- ``docker build``: Used to build the container app image.
- ``docker login``: Used for login into the ACR. To login, it uses the ACR admin user and password.
- ``docker push``: Used to push the image into the ACR.

The following code snippet shows how we're using the Terraform ``null_resource`` resource to invoke a shell script that runs the docker commands.
```yaml
# Set container image name
locals {
  image_name = "init-app-null-resource-with-docker-commands"
  image_tag = "latest"
}

# Create docker image
resource "null_resource" "docker_image" {
    triggers = {
        image_name = local.image_name
        image_tag = local.image_tag
        registry_uri = data.azurerm_container_registry.acr.login_server
        dockerfile_path = "${path.cwd}/init-app/Dockerfile"
        dockerfile_context = "${path.cwd}/init-app"
        registry_admin_username = data.azurerm_container_registry.acr.admin_username
        registry_admin_password = data.azurerm_container_registry.acr.admin_password
        dir_sha1 = sha1(join("", [for f in fileset(path.cwd, "init-app/*") : filesha1(f)]))
    }
    provisioner "local-exec" {
        command = "./scripts/docker_build_and_push_to_acr.sh ${self.triggers.image_name} ${self.triggers.image_tag} ${self.triggers.registry_uri} ${self.triggers.dockerfile_path} ${self.triggers.dockerfile_context} ${self.triggers.registry_admin_username} ${self.triggers.registry_admin_password}" 
        interpreter = ["bash", "-c"]
    }
}
```
And this is how the Shell script looks like.
```shell
#!/bin/bash
IMAGE_NAME="$1"
IMAGE_TAG="$2"
REGISTRY_URI="$3"
DOCKERFILE_PATH="$4"
DOCKERFILE_CONTEXT="$5"
REGISTRY_USERNAME="$6"
REGISTRY_PASSWORD="$7"

docker build -t $REGISTRY_URI/$IMAGE_NAME:$IMAGE_TAG -f $DOCKERFILE_PATH $DOCKERFILE_CONTEXT
docker login $REGISTRY_URI -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD 
docker push $REGISTRY_URI/$IMAGE_NAME:$IMAGE_TAG
```

- **What's the advantage of this scenario over the previous ones?**

In my opinion, there is no advantage; I consider this the least preferable of the three scenarios. I would opt to run the other two scenarios before considering this one.
