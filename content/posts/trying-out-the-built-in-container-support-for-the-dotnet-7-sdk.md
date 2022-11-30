---
title: "Trying out the built-in container support for the .NET SDK"
date: 2022-11-30T10:01:13+01:00
tags: ["dotnet", "docker", "containers", "aws", "devops", "azure"]
description: "A few months ago the built-in container support for the .NET SDK was announced. In this post I'll put this feature to test, I'll try to migrate from an application that contains a rather complex Dockerfile to a new version that has no Dockerfile and instead uses the container support feature."
draft: false
---

> **Just show me the code!**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/trying-out-the-built-in-container-support-for-the-dotnet-7-sdk).

A few months ago, the built-in container support for the .NET SDK was announced. This feature allows us to containerize our application using the ``dotnet publish`` command, with no need of having to write a  ``Dockerfile``.

To make it work we just have to add a series of properties in the application project file (.csproj), install the ``Microsoft.NET.Build.Containers`` package and run the ``dotnet publish`` command, the output artifact will be a container image of our app.

I've been wanting to try this new feature for quite a while, but I don't want to use it with a simple "Hello World" .NET app, because I know that it will work well with it.    
Instead, **I decided that I'll try to migrate an app that has a rather "complex" ``Dockerfile`` to a new version that has no ``Dockerfile`` and uses the built-in container support feature**.

# **Application & Dockerfile**

The application is a BookStore API built using .NET 7. It allows us to do the following actions:

- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and delete orders.

But the application per se doesn't matter at all, the important part of the app is the ``Dockerfile``. 

> Remember that the purpose of this post is to evaluate if we can move from an app that uses a complex ``Dockerfile``  to a _"dockerfile-less"_ app that uses the container support for .NET SDK.


The first step is to know what the BookStore API ``Dockerfile`` is doing, so we can replicate those same features with the container support for .NET.   

The BookStore API ``Dockerfile`` contains the following features:
- It uses a multi-stage build.
- It uses 2 private platform images as base images. Those images are hosted on a private ``AWS ECR`` repository.
- The API references a few private NuGet packages hosted on my private ``Azure Artifacts NuGet feed``, which means that the ``Dockerfile`` has to make use of the [Azure Artifacts Credential Provider](https://github.com/microsoft/artifacts-credprovider) when trying to run the ``dotnet restore`` command.
- The application is published using the ``PublishTrimmed`` feature. This feature removes unnecessary framework components from the resulting artifact.
- The app is published using the ``self-contained`` mode.
- The container uses a ``non-root user`` to run the API _(This is done in the base image, not in the application Dockerfile. More info in the next section)_.
  
The next code snippet shows how the ``Dockerfile`` looks like.
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

As I stated in the previous section, the BookStore API ``Dockerfile`` uses a multi-stage build with a pair of private platform images as base images.

> But, what`s a platform image?

When talking about containers security, one of the best practices is to use your own platform images, those platform images are the base for your company applications.   

The point of having a platform image instead of directly using a base image is to ensure that the resulting containers are hardened according to any corporate policies before being deployed to production.

![platform-images](/img/platform-images.png)

The BookStore API  ``Dockerfile`` uses a multi-stage build to create the container image, but the container support feature works a little bit different, when you execute  the ``dotnet publish`` command instead of using a multi-stage approach it creates the application artifact locally (on your machine, outside of Docker) and afterwards the artifact is copied inside an image.

The BookStore API ``Dockerfile`` uses those 2 platform images:
- ``sdk:7.0-bullseye-slim-1.0.0``
- ``runtime-deps:7.0-bullseye-slim-1.0.0``

But the container support for .NET SDK only needs a single base image because the build step is done outside of Docker, which means that the only image we need is the one that contains the runtime dependencies: ``runtime-deps:7.0-bullseye-slim-1.0.0``

Let's take a look at what the ``runtime-deps:7.0-bullseye-slim-1.0.0`` image contains:

- A user named ``devsecops`` is created with the ``adduser`` command (The ``gecos`` argument prevents the system from asking for additional details).
- The ``chown`` command is used to set the ``devsecops`` user as owner of the ``/app`` directory.
- Non-root users are not allowed to allocate ports below ``1024``, with the ``ASPNETCORE_URLS`` environment we're settings the port ``8080`` as the running port.

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

Now that we know how the ``Dockerfile`` on the BookStore API looks like, it's time to delete it from the solution and start using the container support for .NET.

## **1. Install the ``Microsoft.NET.Build.Containers`` NuGet package.**

This NuGet is reponsible to enable the container support features.

Right now (11/29/2022) the latest version of this package is: 
- _0.3.0-alpha.16_    

This version is not available on nuget.org, you can get it from its [GitHub page](https://github.com/dotnet/sdk-container-builds).


## **2.Configure the csproj file**

The way to configure the container support for .NET SDK is through a set of MSBuild properties that can be placed into the application project (.csproj) file.

The container support properties we're going to need are the following ones:
- ``ContainerBaseImage``
	- Specifies the base image used for your resulting image. 
	- The value is going to be: ``823934831816.dkr.ecr.eu-west-1.amazonaws.com/runtime-deps:7.0-bullseye-slim-1.0.0``
- ``ContainerImageName``
	- This property controls the name of the image itself.
	- If a container image is not specified, it will use the ``AssemblyName`` of the project. In our case we're going to use: ``bookstore-webapi``
- ``ContainerImageTags``
	- This property controls the tags that are generated for the image.
	-  If a container tag is not specified, it will use the ``Version`` of the project. In our case we're going to use: ``latest``
- ``ContainerPort``
	- This attribute  adds TCP or UDP ports to the list of known ports for the container, it is the equivalent of the ``EXPOSE`` command on the ``Dockerfile``.
	- This attribute does absolutely nothing, it is an informative only attribute.
	- The value is going to be:  ``<ContainerPort Include="8080" Type="tcp" />``
- ``PublishProfile``
	- For containerizing a webapp the property ``PublishProfile=DefaultContainer`` is required.

The Bookstore API ``Dockerfile`` is also using the ``PublishTrimmed`` attribute, to reduce the size of the resulting artifact, and the ``self-contained`` attribute to publish the app as an executable, to mimick the same behaviour we're going to add them on the project (csproj) file:

- ``SelfContained``
- ``PublishTrimmed``

Also the ``RuntimeIdentifier`` attribute is required. This attribute is used to identify the target platforms where the application needs to run, in this case the value will be ``linux-x64`` because we're building a Linux container.

Instead of putting the ``SelfContained``, ``RuntimeIdentifier`` and ``PublishTrimmed`` properties on the project file, we can aso specify them when executing the ``dotnet publish`` command, like this: ``dotnet publish --self-contained true --runtime linux-x64 /p:PublishTrimmed=true``   

If you decide to put those properties on the project file (.csproj) you must know that you won't be able to build the app with Visual Studio on a Windows machine, mainly because you're targeting Linux. Furthermore, the build and debug process on Visual Studio it's going to take longer than usual because it needs to trim and self-contain the application artifact. If any of this bothers you, remove those properties from the csproj and specify them when executing the ``dotnet publish`` command.

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

Now it's time to run the ``dotnet publish`` command that creates the container image, but this application is using a few private resources, to be more precise:

- The app has a few references to private packages hosted on my private ``Azure DevOps`` feed.
- The app uses the ``runtime-deps:7.0-bullseye-slim-1.0.0`` as base image. This image is hosted on my private ``AWS ECR`` repository.

So, before running the ``dotnet publish`` command  I have to setup the ``AWS`` and ``Azure DevOps`` credentials on my local machine to allow the .NET CLI to retrieve them.

- To allow the ``dotnet publish`` command to pull the private image from ``ECR``, I'm going to make a ``docker login`` command and use a [Named Profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) alongside the ``AWS_PROFILE`` environment variable.
- To allow the .NET CLI to restore the private packages from my private ``Azure DevOps`` feed, I'm going to place a ``NuGet.Config`` with the proper credentials on [my computer](https://learn.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior).


After settings the credentials properly, let's run the ``dotnet publish`` command and the BookStore API container image is  created successfully.

![container-tools-sdk-create-image](/img/container-tools-sdk-image-size.png)

If you take a look at the picture above, you'll see that the image created with the ``dotnet publish`` command (bookstore-webapi) is exactly the same size as the one created using the ``Dockerfile`` (bookstore.api).

# **Building a container image using Azure Pipelines**

We know from the previous section that moving from the BookStore API ``Dockerfile`` to a "dockerfile-less" app that uses the built-in container support for .NET is doable on my local machine, but is it feasible on a CI/CD environment?

For this test I'm going to use ``Azure Pipelines``, if you want to use any other CI/CD runner (GitHub Actions, Bitbucket, etc.) it should be relatively easy to switch, because the idea behind it is the same for any runner.

The pipeline needs to do the same steps we have done on our local machine:
- Restore the application NuGet packages. 
  - The app contains some private reference, and those packages are hosted on a private ``Azure DevOps`` feed, which means that the pipeline task has to use the ``vstsFeed`` attribute to specify which feed will be used when restoring the packages.
- Login into the docker registry
- Execute the ``dotnet publish`` command.
- And finally, let's try to push the image into an ``AWS ECR`` repository, why not?

The container support for .NET has the ``ContainerRegistry`` attribute, which allow us to push the resulting image into a registry, if this value is not present on the csproj then the image gets stored in the local Docker daemon.    

Right now the ``ContainerRegistry`` attribute doesn't work with ``AWS ECR``. If you need to push the resulting image into an ``AWS ECR`` repository you have to store it in you local Docker machine and then push it onto the repository using the ``docker push`` command.

Here's how the resulting ``Azure Pipeline`` looks like:
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

We have successfully created the BookStore API container image using the container support for .NET SDK, but does it work? 

Let's run it using the  ``docker run -d -p 5001:8080 bookstore-webapi`` command and then let's send a request to ``https:\\localhost:8080\api\orders`` using cURL, and it doesn't work...

The container appears to be running properly, but what it seems weird is that Kestrel is listening on port ``80`` instead of port ``8080`` that it was the one we defined on our base image using the ``ASPNETCORE_URLS`` environment variable.

![container-tools-sdk-starting-port](/img/container-tools-sdk-starting-port.png)


If we open a Shell session inside the container using the ``docker exec -it <container-id> sh`` command, it seems that the ``root`` user owns the ``/app`` folder, but have created a non-root user called ``devsecops`` who should own this directory.

![container-tools-sdk-pernissions](/img/container-tools-sdk-pernissions.png)

It seems that the container support for .NET takes a few concessions when creating an image, there are certain values that grt overriden like the running port or the users and permissions, which represents a problem if you're running an app that listens on a non-80 port or uses a non-root user. 

In fact, if we try to run the BookStore API on port 80 using the ``docker run -d -p 5001:80 bookstore-webapi`` command, it app works perfectly.

- Taking a look at [GitHub](https://github.com/dotnet/sdk-container-builds/issues/165), it seems that there is an ongoing issue with users and permissions.




# **Closing thoughts**

The purpose of this post was to evaluate if we could move from an app that was using a ``Dockerfile``  to a "dockerfile-less" app that uses the container support for .NET.

My overall impressions is that the container support for .NET works pretty well.   

Restoring packages from a private feed on a .NET app is always a pain in the ass when building an image using a multi-stage approach, but the fact that the container support builds the artifact outside of docker makes the process more palatable.

Concerning the ability to push and pull images to and from a private registry, it is very good news that the container support is able to authenticate to a container registry, which allows us to use our own private base images and not have to rely on public images from Microsoft.   
The fact that the image cannot be pushed to ``AWS ECR`` is a shame, but is not a deal breaker at all, you can always store it on your local docker daemon and push it afterwards using the ``docker push`` command.

Right now, the most well-known constraint when using the container support for .NET SDK is the fact that it can't execute ``RUN`` commands, for this post this was not important because the BookStore API ``Dockerfile`` didn't need to use it, but that's something to keep in mind.   
A possible solution if you're using a ``RUN`` command on your ``Dockerfile`` and you want to start using the container support is to move the command into a base image.

Being able to use the container support for .NET SDK on a CI/CD enviroment is feasible and quite easy to do it, and in this post we have demonstrated that it is possible to do it.

So far, everything related to the container support seems positive, but the fact that you cannot run a container with a non-root user is a deal breaker from me, I cannot fathom any excuse for why you'd only allow to create container that can be run by a root user.

So, if you're in a big enterprise we're security is paramount and everything gets scanned and deeply reviewed, maybe it's not yet the time to move away from the ``Dockerfile``.
