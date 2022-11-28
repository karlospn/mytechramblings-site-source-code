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

> But, what`s a platform image?

When talking about containers security on the enterprise one of the best practices is to use your own platform images, those platform images are the base for your company applications.

<add-img>

The point of having a platform image instead of directly using a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.

The BookStore API  ``Dockerfile`` uses a multi-stage build to create the app image, but the ``built-in container support for .NET7 SDK`` works a little bit different, instead of using a multi-stage approach it creates the application artifact locally (on your machine, outside of Docker) and afterwards the container image gets created using the local artifact.

The BookStore API ``Dockerfile`` uses a those 2 platform images:
- ``sdk:7.0-bullseye-slim-1.0.0``
- ``runtime-deps:7.0-bullseye-slim-1.0.0``

But as I have stated in the previous paragraph, the container support for .NET7 SDK only needs a single image because the build step is done outside of Docker, which means that the only image we need is the one that contains the runtime: ``runtime-deps:7.0-bullseye-slim-1.0.0``

Let's take a look at how we are building the ``runtime-deps:7.0-bullseye-slim-1.0.0`` image:

- A user named ``devsecops`` is created with the ``adduser`` command (The ``gecos`` argument prevents the system from asking for additional details).
- The ``chown`` command is used to set the ``devsecops`` user as owner of the ``/app`` directory.
- Non-root users are not allowed to allocate ports below ``1024``, the ``ASPNETCORE_URLS``environment is setting the running port to be the ``8080``.

```yaml
FROM mcr.microsoft.com/dotnet/runtime-deps:7.0-bullseye-slim

# Set internal version
ENV INTERNAL_VERSION=11202022.1

# Set non-root user
RUN adduser --disabled-password \
  --home /app \
  --gecos '' devsecops && chown -R devsecops /app

# Set the default user.
USER devsecops

# Non-user root cannot start on port 80
ENV ASPNETCORE_URLS=http://+:8080
```

# **Implementation**

Now that we know how the ``Dockerfile`` on our app looks like, it's time to remove it from the solution and start using the built-in container support for .NET7 SDK.

## **1. Install the ``Microsoft.NET.Build.Containers`` NuGet package.**

Right now (11/28/2022) the latest version of this package is:
- 0.3.0-alpha.16

This version is not available on nuget.org, you can get it from its [GitHub page](https://github.com/dotnet/sdk-container-builds)

For this post I have uploaded it on my private Azure DevOps artifact feed.

## **2.Configure the csproj**

The way to configure the ``built-in container support for .NET7 SDK`` is through a set of MSBuild properties that can be set into the application project (.csproj) file.

The container support properties we're going to need are the following ones:
- ``ContainerBaseImage``
	- It controls the image used as the basis for your resulting image. 
	- The value is going to be: ``823934831816.dkr.ecr.eu-west-1.amazonaws.com/runtime-deps:7.0-bullseye-slim-1.0.0``
- ``ContainerImageName``
	- This property controls the name of the image itself.
	- If a container image is not specified, it will use the ``AssemblyName`` of the project. In our case we're going to use: ``bookstore-webapi``
- ``ContainerImageTags``
	- This property controls the tag that is generated for the image.
	-  If a container tag is not specified, it will use the ``Version`` of the project. In our case we're going to use: ``latest``
- ``ContainerPort``
	- This attribute  adds TCP or UDP ports to the list of known ports for the container. It is the equivalent of the ``EXPOSE`` command on the ``Dockerfile``.
	- This attribute does absolutely nothing, it is an informative attribute.
	- The value is going to be:  ``<ContainerPort Include="8080" Type="tcp" />``
- ``PublishProfile``
	- For containerizing a webapp the property ``PublishProfile=DefaultContainer`` is required.

The Bookstore API ``Dockerfile`` is using the ``PublishTrimmed`` attribute to reduce the size of the resulting artifact, and also uses the ``self-contained`` attribute to publish the app as an executable, to repeat the same behaviour we're going to add a few more build properties to our project file:

- ``SelfContained``
- ``PublishTrimmed``
- ``RuntimeIdentifier``


The ``SelfContained``, ``RuntimeIdentifier`` and ``PublishTrimmed`` properties can be placed on the project file or we specify them when executing the ``dotnet publish`` command, like this: ``dotnet publish --self-contained true --runtime linux-x64 /p:PublishTrimmed=true`` command.   

If you decide to place those properties on the project file (.csproj) you should know if you're running on a Windows machine won't be able to build the app using Visual Studio because you're targeting Linux, also the build and debug process it's going to take longer because it needs to apply the trimming and self-contain process.

Here's how the project file will end up looking like:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
	<TargetFramework>net7.0</TargetFramework>
	<ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
	
  <PropertyGroup>
	<RuntimeIdentifier>linux-x64</RuntimeIdentifier>
	<PublishProfile>DefaultContainer</PublishProfile>
	<SelfContained>true</SelfContained>
	<PublishTrimmed>true</PublishTrimmed>
	<ContainerImageName>bookstore-webapi</ContainerImageName>
	<ContainerBaseImage>823934831816.dkr.ecr.eu-west-1.amazonaws.com/runtime-deps:7.0-bullseye-slim-1.0.0</ContainerBaseImage>
	<ContainerImageTags>latest</ContainerImageTags>
  </PropertyGroup>

  <ItemGroup>
	<ContainerPort Include="8080" Type="tcp" />
  </ItemGroup>

  <ItemGroup>
  	<PackageReference Include="Microsoft.NET.Build.Containers" Version="0.3.0-alpha.16" />
	<PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.0" />
	<PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
	<PackageReference Include="MyOwn.EmailService" Version="1.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\BookStore.Domain\BookStore.Domain.csproj" />
    <ProjectReference Include="..\BookStore.Infrastructure\BookStore.Infrastructure.csproj" />
  </ItemGroup>

</Project>
```

## **3.Create the container**

Now it's time to run the ``dotnet publish`` command that creates the container image, but this application is using a series of private resources. To be more precise:

- The app has a few references to private packages hosted on my private ``Azure DevOps`` feed.
- The app uses the ``runtime-deps:7.0-bullseye-slim-1.0.0`` as base images. This image is hosted on my private ECR repository.

So, before running the ``dotnet publish`` command,  I have to setup the ``AWS`` and the ``Azure DevOps` credentials on my local machine, so the .NET CLI is capable to retrieve them.

- To allow the ``dotnet publish`` command to pull the private image from ``ECR``, I'm going to make a ``docker login`` command and use a [Named Profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) alongside the ``AWS_PROFILE`` environment variable to set the AWS credentials.
- To allow the .NET CLI to restore the private package from my private ``Azure DevOps`` feed, I'm going to place a ``NuGet.Config`` with the proper credentials on [my computer](https://learn.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior).


After settings the credentials, let's run the ``dotnet publish`` command, and it just works!

<add-img>

If you take a look at the picture above, you'll see that the image created with the ``dotnet publish`` command is exactly the same size as the one created using the ``Dockerfile``.

# **Building a container image using Azure Pipelines**

We know from the last section that moving from the BookStore API ``Dockerfile`` to a "docker-less" app that uses the ``built-in container support for .NET7 SDK`` is doable on my local machine, but is doable on a CI/CD environment?

Let's try it, I'll be using ``Azure Pipelines``, but the concepts are the same for any CI/CD runner, so you can use whatever you prefer: Github Actions, Bitbucket, etc.

The pipeline has to do the same steps we have done on our local machine:
- Restore the application NuGet packages. Remember that the app contains a few references to private packages hosted on my private ``Azure DevOps`` feed. On the pipeline you can use the ``vstsFeed`` attribute to specify which feed to use when restoring the packages.
- Login into the docker registry
- Execute the ``dotnet publish`` command.
- Push the image into an ``AWS ECR`` repository.

> You can use the ``ContainerRegistry`` attribute from the ``built-in container support for .NET7 SDK`` to push the resulting image into a registry, if this value is not present on the csproj the image is stored in the local Docker daemon.    
> Right now the ``ContainerRegistry`` attribute doesn't work with AWS ECR. If you need to push the resulting image into an ECR repository you have to store it in you local Docker machine and then push it onto ECR using the ``docker push`` command.

Here's how the resulting pipeline looks like:
```yaml
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  awsAccount: '823934831816'
  region: 'eu-west-1'

steps:
- task: UseDotNet@2
  displayName: 'Install .NET Core sdk 7.x'
  inputs:
    packageType: 'sdk'
    version: '7.x'

- task: DotNetCoreCLI@2
  displayName: 'Restore packages'
  inputs:
    command: 'restore'
    projects: '**/*.csproj'
    feedsToUse: 'select'
    vstsFeed: 'a4c874c8-2ab9-4a3a-aa79-6f007f55ee6f'
    
- task: AWSShellScript@1
  displayName: 'Build container and publish to AWS ECR'
  inputs:
    awsCredentials: 'aws-dev'
    regionName: '$(region)'
    scriptType: 'inline'
    inlineScript: |
      aws ecr get-login-password --region $(region) | docker login --username AWS --password-stdin $(awsAccount).dkr.ecr.$(region).amazonaws.com
      dotnet publish $(Build.SourcesDirectory)/src/BookStore.WebApi/BookStore.WebApi.csproj --no-restore
      docker tag bookstore-webapi:latest $(awsAccount).dkr.ecr.$(region).amazonaws.com/bookstore.webapi:latest
      docker push $(awsAccount).dkr.ecr.$(region).amazonaws.com/bookstore.webapi:latest
```

# **Testing the resulting container image**

We have created a container image using container support for .NET7 SDK, but does this container runs properly? 

Let's run the ``docker run -d -p 5001:8080 bookstore-webapi`` command and it doesn't work out...

The first unusual thing is that the container is running properly and it works fine, but is listening on port ``80`` instead of the ``8080`` we defined on our base image.

<add-img>

Let's go inside the container using the ``docker exec -it <container-id> sh`` command, it seems that the ``root`` user is the one who owns the ``/app`` folder, but have created a non-root user called ``devsecops`` who should own this folder...

<add-img>

Maybe the "ContainerBaseImage" property is not working properly and we're not using our platform image?   

In the ``runtime-deps:7.0-bullseye-slim-1.0.0`` platform image we have built previously, we have set this environment variable ``ENV INTERNAL_VERSION=11202022.1``.    
Let's go inside the container using the ``docker exec -it <container-id> sh`` command and the ``printenv`` command let0s take a look at the container environment variables.

<add-img>

The ``INTERNAL_VERSION=11202022.1`` environment variable is properly set, which means that this container is using our platform image as base image, but the starting port and the user permissions seems to have been overriden.

Taking a look at GitHub seems that there is an ongoing issue when 
- https://github.com/dotnet/sdk-container-builds/issues/165


# **Closing thoughts**

The purpose of this post was to evaluate if we could move from an app that was using a rather complex ``Dockerfile``  to a "docker-less" app that uses the container support for .NET 7 SDK to generate the container image.

My overall impression is that it works pretty well, restoring packages from a private feed is always a pain in the ass in Docker, but the fact that the container support builds the artifact outside of docker makes the process more palatable.

The fact the the container support is able to authenticate to a container registry allow us to use our own private base images and not have to rely on public images from Microsoft.   
The fact that the image cannot be pushed to ECR using the container support is a shame, but is not a deal breaker at all, because you can always store it on your local docker daemon and push it afterwards using the ``docker push`` command.

The most well-known constraint when using the container support for .NET 7 SDK is the fact it can't execute ``RUN`` commands, in this test this was not important at all because our ``Dockerfile`` doesn't need to use the ``RUN`` command, but that's something you must keep in mind.   
A possible solution if you're using some ``RUN`` commands on your ``Dockerfile`` and you want to start using the container support is to move those ``RUN`` commands into a base image.

Being able to use the container support on a CI/CD enviroment was also a very important feature, and in this post we demonstrate that it was possible to do it.

So far, everything related to the container support seems positive, but the fact that you cannot run a non-root container is a deal breaker from me, I cannot fathom any excuse for why you'd want to only allow to run root containers.    

So, if you're in a big enterprise we're security is paramount and everything gets scanned and deeply reviewed maybe it's not the time to move away from the ``Dockerfile``.
