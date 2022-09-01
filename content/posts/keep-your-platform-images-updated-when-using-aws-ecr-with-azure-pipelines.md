---
title: "Keep your .NET platform images up to date using AWS ECR with Azure Pipelines"
date: 2022-08-23T15:36:41+02:00
tags: ["aws", "dotnet", "devops", "ecr", "containers"]
description: "When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images will be the base for your company applications. And in this post I'm going to show you an opinionated implementation of how to automate the creation and update of your .NET platform images."
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/keep-your-platform-images-updated-when-using-aws-ecr-and-azure-pipelines).

In this post I'm going to show you an opinionated implementation of how to automate the creation and update of your .NET platform images.

This implementation is going to be built to work solely with **AWS Elastic Container Registry** (AWS ECR) as image registry and **Azure Pipelines** as orchestrator for the creation or update of the platform images.  

But first, let's talk a little bit about what's a platform image.

# What's a platform image?

When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images will be the base for your company applications.

![platform-images-diagram](/img/platform-images.png)

The point of having and using a platform image instead of a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.

If your enterprise applications need some kind of software baked into the image to run properly, it is far better to install it only on the platform image, instead of having to install it in each and every one of the application images, this way it is far easier to roll updates.

For dotnet there are a few official docker images avalaible on the Microsoft registry (https://hub.docker.com/_/microsoft-dotnet/), but we don't want to use those images directly on our enterprise applications. Instead of that we are going to use those official images from Microsft as a base image to build a platform image that is going to be own by our company.   

The Microsoft official dotnet images are not inherently bad and often include many security best practices, but a more secure way is to not rely solely on third party images, because you lose the ability to control scanning, patching, and hardening across your organization. The recommended way is using these images as the base for building any of your company platform images.

# **Create and update a dotnet platform image**

Create a new platform image is an easy, but you also want to keep the image up to date.   
Every time the base image gets a new update from Microsoft you'll want to update your platform image, because the latest version it's usually the most secure.

The Microsoft image update policy for dotnet image is the following one:
- The .NET base images are updated within 12 hours of any updates to the underlying OS (e.g. debian:buster-slim, windows/nanoserver:ltsc2022, buildpack-deps:bionic-scm, etc.).
- When a new version of .NET (includes major/minor and servicing versions) gets released the dotnet base images get updated.

The .NET base images are updated quite freqüently, which means that automating the update of our platform image is paramount.

In this section we have talked about creating a new platform image and also the importance of keep it up to date when a new update on the base image is avalaible, but it is also possible that you might want to change something on an existing platform image (for example, install some new software, update some existing one, set some permissions, etc) and after doing that you will want to create a new version of the platform image right away.

I see 3 main things we'll need to implement to automate the creation/update of a dotnet platform image:
- Creation of a new platform image.
- Update of an existing platform image because the base image has a new updated version available.
- Update of an existing platform image because we have modified something and want to create a new version of the image.

# **dotnet platform image creation/update automated process diagram**

The following diagram shows the necessary steps we're going to build to automate the creation or update of a platform image.  

The platform image creation/update process is going to be executed on an **Azure DevOps Pipeline**.   
Every platform image will have its own Azure DevOps pipeline.

![pipeline-diagram](/img/update-platform-images-pipeline.png)

The pipeline gets triggered on a scheduled basis (every Monday at 6:00 UTC), it uses a scheduled trigger because we want to periodically poll the Microsoft container registry (https://mcr.microsoft.com/) to check if there is any update available for the base image.

When trying to update an image if there is NO new update available on the base image then the pipeline just ends there.   

If there is a new updated base image on the Microsoft registry then the pipeline will create a new version of the platform image, test it, store it into ECR and notify the update into a Teams channel.

Also if you want to change something on an existing platform image (for example, install some new software, update some existing one, set some permissions, etc) you will want to create a new version of the platform image right away, you can do it setting the ``force_update`` pipeline parameter to ``true``, that parameter will skip the Microsoft container registry update check and go straight into creating a new platform image.

## **How to know if there is a new updated base image?**

If you're using Azure Container Registry (ACR) as your container registry you can use ACR Tasks to automate the creation of a new platform image when a container's base image is updated.
- _More info here about it: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-base-image-update_

But this kind of functionality does not exist for AWS ECR, so you'll need to build something on the side.

There are some third party tools like Diun or image-watch that allows us to receive notifications when a Docker image is updated on a Docker registry, but there is a much easier approach that using those tools:
- Use Azure Pipelines (or any other CI/CD service like GitHub Action) with a Cron Trigger that runs a pipelines thats checks if the base image has been updated.

But, how do we know if a dotnet base image has been updated?     
We can query the Microsoft Artifact Registry (https://mcr.microsoft.com) for any dotnet image to obtain when was pushed last time.   
For example, if we want to know when the "mcr.microsoft.com/dotnet/runtime:6.0-bullseye-slim" image whas pushed last image we can go fetch that information from here: "https://mcr.microsoft.com/api/v1/catalog/dotnet/runtime/tags".

From this point forward we can fetch the last time our platform image was pushed into AWS ECR and if this value is inferior then it means we're not using the most up to date base image and a new platform image needs to be built.

## **What's the test step all about?**

You'll see a "Test Platform Image" if you take a look at the diagram from above, but what does it mean?

When updating a platform image we need to validate that the updated version are not going to break anything that could potentially affect our enterprise applications.   

The pipeline test step validates that you can run the platform image version that you're building in a real application without any unexpected error.

## **What tagging strategy are we going to use?**

A tagging strategy is important for managing multiple versions of containers and make consuming images easier across your organization.

The platform images will be tagged using the upstream version of the platform with some minor tweaks.

Let's take a look at how Microsoft tags their images:

- _{dotnet-version}-{os-version}-{architecture-os}_
 
 Here's a few example of how the .NET runtime image is tagged:

- 6.0.8-bullseye-slim-amd64
- 6.0-bullseye-slim-amd64
- 6.0.8-bullseye-slim
- 6.0-bullseye-slim
- 6.0.8
- 6.0

For my platform images I'm going to use a similar approach

- _{dotnet-version}-{os-version}-{versioning}_

And it will look like this:

- 6.0-bullseye-slim-1.0.0  
- 6.0-bullseye-slim.1.1.0  

A version is added at the end of the tag so if there is the necessity to use a previous platform image we can use a specific version.







# **Building the dotnet platform image creation/update automated process.**

## **Key points**
From the previous section I can extract the following key points to keep in mind when building this solution:

- Platform images are going to be stored in AWS ECR.
- Azure Pipelines to run the process of creation/update the platform images.
- An Azure Pipeline scheduled trigger will be used to periodically check if any dotnet base image contains a new update.
- The ``Dockerfile`` for any platform image will be hosted on Azure DevOps Repos.
- Every time a new version of a platform is created
- Every time a new version of a platform is created a notification to a Teams Channel needs to be sent.

## **Repository structure**

## **Building the pipeline**