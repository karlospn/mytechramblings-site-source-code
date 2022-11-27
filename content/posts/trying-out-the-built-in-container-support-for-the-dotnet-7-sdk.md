---
title: "Trying out the built-in container support for the .NET 7 SDK"
date: 2022-11-22T13:01:13+01:00
tags: ["dotnet", "containers", "aws", "devops", "azure"]
description: "A few months ago the built-in container support in the .NET 7 SDK was announced. This feature allows us to containerize our application just using the dotnet publish command without no need of having to write a Dockerfile. In this post I'll try to migrate from an app that uses a complex Dockerfile to a 'docker-less' app that uses the container support for .NET 7 SDK."
draft: true
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/trying-out-the-built-in-container-support-for-the-dotnet-7-sdk).

A few months ago the built-in container support in the .NET 7 SDK was announced. This feature allows us to containerize our application just using the ``dotnet publish`` command without no need of having to write a ``Dockerfile``.

With this approach instead of using a ``Dockerfile`` we just have to add a series of properties in the application project file (.csproj), install the ``Microsoft.NET.Build.Containers`` NuGet package, run the ``dotnet publish`` command and the output artifact will be a containerized image of your application.

I've been wanting to try this new feature for quite a while, but I don't want to use it with a simple "Hello World" .NET app, because I know that it will work well with it. 

Instead of that in this post **I'll try to migrate from an app that uses a complex ``Dockerfile``  to a "docker-less" app that uses the container support for .NET 7 SDK**, and see how it performs.

# **Application & Dockerfile**

The application is a BookStore API built using .NET6. It allows us to do the following actions:

- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and delete orders.

But the application per se doesn't matter at all, the important part of the application is the ``Dockerfile``. 

Remember that the purpose of this post is to evaluate if we can move from an app that uses a complex ``Dockerfile``  to a "docker-less" app that uses the container support for .NET 7 SDK.

The BookStore API ``Dockerfile`` uses a multi-stage build, and contains the following features:
- It uses a pair of private platform images as base images. Those images are hosted on a private ``AWS ECR`` repository. Using a platform image instead of using directly the public Microsoft images is a good security practice and also quite common in the enterprise.
- The app contains a few references to private packages hosted on a private ``Azure Artifacts NuGet feed``, which means that the ``Dockerfile`` uses the [Azure Artifacts Credential Provider](https://github.com/microsoft/artifacts-credprovider) to restore the NuGet packages. 
  - The ``Azure Artifacts Credential Provider`` uses an ``Azure DevOps Personal Access Token`` to restore the NuGet packages.
- To optimize the application size, the application must be published using the .NET trim feature. This feature removes unnecessary framework components from the resulting artifact.
- The application must be published using the ``self-contained`` mode.
- The application runs using a ``non-root user``. _(This is done in the platform base image)_
  
And here's how the ``Dockerfile`` looks like.
```yaml
FROM 823934831816.dkr.ecr.eu-west-1.amazonaws.com/sdk:7.0-bullseye-slim-1.0.0 AS build-env
WORKDIR /app

# Get Azure DevOps PAT from args
ARG AZDO_PAT

# Setup cred-provider
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS {\"endpointCredentials\": [{\"endpoint\":\"https://pkgs.dev.azure.com/cponsn/_packaging/my-very-private-feed/nuget/v3/index.json\", \"username\":\"build\", \"password\":\"$AZDO_PAT\"}]}

# Copy everything and restore packages
COPY . ./
RUN dotnet restore -s "https://pkgs.dev.azure.com/cponsn/_packaging/my-very-private-feed/nuget/v3/index.json" \
    -s "https://api.nuget.org/v3/index.json" \
	--runtime linux-x64	

# Build project
RUN dotnet build "./src/BookStore.WebApi/BookStore.WebApi.csproj" \ 
    -c Release \
	--runtime linux-x64 \ 
	--self-contained true \	
	--no-restore


# Publish app
RUN dotnet publish "./src/BookStore.WebApi/BookStore.WebApi.csproj" \
	-c Release \
	-o /app/publish \
	--no-restore \ 
	--no-build \
	--self-contained true \
	--runtime linux-x64 \
	/p:PublishTrimmed=true

# Build runtime image
FROM 823934831816.dkr.ecr.eu-west-1.amazonaws.com/runtime-deps:7.0-bullseye-slim-1.0.0

# Expose port
EXPOSE 8080

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Set entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```

# **Platform images**

As I said before, the BookStore ``Dockerfile`` uses a multi-stage build with a pair of private platform images as base images.

But, what`s a platform image?

When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images are the base for your company applications.

<add-img>

The point of having a platform image instead of directly using a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.

The BookStore API  ``Dockerfile`` uses a multi-stage build to create the app image, but the ``built-in container support for .NET7 SDK`` works a little bit different, instead of using a multi-stage approach it creates the application artifact locally (on your machine, outside of Dodker) and afterwards a container image gets created using the local artifact.

Let's take a look at how the ``runtime-deps:7.0-bullseye-slim-1.0.0`` platform images looks like:

```yaml

```



# **Implementation**

Now that we know how the ``Dockerfile`` on our app looks like, it's time to delete it and start moving the functionality to the built-in container support for .NET7 SDK.

# **Building a container using Azure Pipelines**

# **Testing the resulting container image**

# **Final thoughts**