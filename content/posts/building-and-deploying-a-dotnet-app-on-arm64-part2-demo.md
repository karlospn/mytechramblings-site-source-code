---
title: "Building and deploying a .NET 8 App on an ARM64 processor using Azure Pipelines and AWS ECS Fargate. Part 2: Demo"
date: 2024-03-01T09:42:30+01:00
tags: ["dotnet", "containers", "azure", "aws", "devops", "arm64"]
description: "In this two-part series, I’m going to show you how to build and deploy a .NET 8 app container image that targets an ARM64 processor. In part 2, we will attempt to perform an end-to-end process. We will build a .NET 8 API, containerize the app targeting an ARM64 processor using Azure Pipelines, and finally deploy it on AWS ECS Fargate.    
Additionally, we will conduct a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container."
draft: true
---

> This is a two-part series post.
> - **Part 1**: Key concepts about how to build multi-platform images.
> - **Part 2**: A practical example of how to build a container image targeting an ARM64 processor using Azure Pipelines and how to deploy it on AWS ECS Fargate.    
It will also include a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container.

In the first part of this blog post, we discussed how to create a multi-platform image, how multi-platform .NET images work, and which options are available when we want to build a multi-platform image (emulation, cross-compilation, or using a host with the target architecture).

Now, it's time to build an end-to-end process. Let's attempt to build a container image targeting ARM64 using a CI runner, deploy the app into an ARM64 machine host, and test that it's working correctly.

I don't have an ARM64 machine at hand right now, so to build the E2E process, I decided to use the following cloud services:

- **AWS ECR** to store the container image.
- **AWS ECS Fargate** to run the API.
- **Azure Pipelines** to build and publish the image into the Amazon Registry.
- **Artillery** to test the API.

Why use AWS instead of Azure? Nothing in particular. In my last posts, I was using Azure, so it might be a nice change of scenery to use AWS.

# **Application**

The application we’re going to use is a BookStore API built using .NET 8.

The API can perform the following actions:

- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and update orders.

This application requires a SQL Server database, but to simplify, we will use EF with an in-memory database.

- You can find the source code [here](https://github.com/karlospn/opentelemetry-metrics-demo)

# **Building the Dockerfile**

Instead of building a single multi-platform Dockerfile, let's make the effort to build two: one multi-platform Dockerfile that uses emulation and another one that uses cross-compilation.

## **Using emulation**

To create a multi-platform Dockerfile that uses emulation, we don't need to do any extra step; just a simple run-of-the-mill Dockerfile will suffice.

Emulation is the easiest option of the three available, because it requires no changes at all to your Dockerfile. The BuildKit automatically detects the secondary architectures that are available and when BuildKit needs to run a binary for a different architecture, it automatically loads it.

The following code snippet shows the Dockerfile with the following features:
- It is a multi-stage Dockerfile: it uses Ubuntu 22.04 as a base image for the build stage and an Ubuntu Chiseled image for the runtime stage.
- The app is published as self-contained, resulting in an application bundle that includes the .NET runtime, libraries, as well as the application itself and its dependencies.

```yml
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy AS build-env
WORKDIR /app

# Copy everything
COPY . ./

# Restore packages
RUN dotnet restore -s "https://api.nuget.org/v3/index.json"

# Build project
RUN dotnet build "./src/BookStore.WebApi/BookStore.WebApi.csproj" \ 
    -c Release \
	--self-contained true

# Publish app
RUN dotnet publish "./src/BookStore.WebApi/BookStore.WebApi.csproj" \
	-c Release \
	-o /app/publish \
	--no-restore \ 
	--no-build \
	--self-contained true

# Build runtime image
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0.0-jammy-chiseled-extra

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Starts on port 8080
ENV ASPNETCORE_URLS=http://+:8080

# Set Entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```

Now, let's try to build the container image on my AMD64 machine. Remember that we're running the ``dotnet build`` command on an AMD64 host machine and we're targeting an ARM64 processor, which means that it will use the [QEMU](https://www.qemu.org/) emulator for building the resulting container image.


Now, let's attempt to build the container image on my AMD64 machine. Remember that we're executing the ``docker build`` command on an AMD64 host machine while targeting an ARM64 processor. This implies that it will utilize the QEMU emulator to construct the resulting container image.

```script
docker build --platform=linux/arm64 -t bookstore-api:arm64-emulation -f src/BookStore.WebApi/Dockerfile .
[+] Building 132.1s (15/15) FINISHED
```
It takes quite some time (and consumes a significant amount of CPU and memory resources), but we eventually end up with a functional ARM64 container image.

## **Using Cross-Compilation**

The idea behind Cross-Compilation is to utilize a multi-stage build Dockerfile. In the build stage, you compile your code for the target architecture, and in the run stage, you configure the runtime to be exported to the final image.

Using the Dockerfile from the previous section as a starting point, there are several modifications needed to make it compatible with Cross-Compilation.

1. Use the pre-defined build argument ``BUILDPLATFORM`` to pin the builder to use the host's native architecture as the build platform. This is necesarry to prevent emulation. 

In simpler terms, we need to add ``--platform=$BUILDPLATFORM`` into the ``FROM`` instruction of the build stage.

2. Add the ``ARG TARGETARCH`` instructions for the build stage, making the ``TARGETARCH`` build arguments available to the commands in this stage.

3. Modify the ``dotnet restore``, ``dotnet build`` and ``dotnet publish`` commands so they generate the application binaries for the target architecture (in our case ``arm64``).    

To accomplish this, we're going to use the ``--arch`` attribute along with the ``TARGETARCH`` argument.

```yml
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0-jammy AS build-env
WORKDIR /app
ARG TARGETARCH

# Copy everything
COPY . ./

# Restore packages
RUN dotnet restore -s "https://api.nuget.org/v3/index.json" \
	-a $TARGETARCH

# Build project
RUN dotnet build "./src/BookStore.WebApi/BookStore.WebApi.csproj" \ 
    -c Release \
	-a $TARGETARCH \ 
	--self-contained true

# Publish app
RUN dotnet publish "./src/BookStore.WebApi/BookStore.WebApi.csproj" \
	-c Release \
	-o /app/publish \
	--no-restore \ 
	--no-build \
	--self-contained true \
	-a $TARGETARCH

# Build runtime image
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0.0-jammy-chiseled-extra

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Starts on port 8080
ENV ASPNETCORE_URLS=http://+:8080

# Set Entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```
Now, let's attempt to build the container image on my AMD64 machine.

```script
$ docker build --platform=linux/arm64 -t bookstore-api:arm64-cc -f src/BookStore.WebApi/Dockerfile .
[+] Building 16.2s (15/15) FINISHED
```
Using cross-compilation, we've reduced the build time 130 seconds to 17 seconds, and the CPU usage from nearly 90% to no more than 20%.

# **Building the container image using Azure Pipelines (and pushing it into ECR)**

In the previous section, we created the Dockerfiles and built the container image on my local machine. Now, instead of using my computer, let's attempt to build it using an Azure Pipelines hosted agent.

## **Using emulation**

Let's start simple. We're using an Ubuntu hosted agent and we're simply running a ``docker build`` command using the ``--platform`` attribute.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker build --platform=linux/arm64 -t bookstore-api:arm64-emu -f src/BookStore.WebApi/Dockerfile .
```

And it throws an error.

```text
#11 [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
#11 0.229 exec /bin/sh: exec format error
#11 ERROR: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 1
------
 > [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json":
0.229 exec /bin/sh: exec format error
------
Dockerfile:8
--------------------
   6 |     
   7 |     # Restore packages
   8 | >>> RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
   9 |     
  10 |     # Build project
--------------------
ERROR: failed to solve: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 1
```

Thi is because the Azure Pipelines linux hosting agent doesn't have QEMU installed, QEMU only comes preinstalled with Docker desktop, if you're using docker engine you need to install it.

To install it, we're going to use the [qemu-user-static](https://github.com/multiarch/qemu-user-static) image. This image everything necessary to run QEMU.

Let's try it again, but installing QEMU first.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
      docker build --platform=linux/arm64 -t bookstore-api:arm64-emu -f src/BookStore.WebApi/Dockerfile .
```

And it throws another error. This time is a segmentation fault error.

```text

#11 [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
#11 22.03   Determining projects to restore...
#11 26.85 Segmentation fault (core dumped)
#11 ERROR: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 139
------
 > [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json":
22.03   Determining projects to restore...
26.85 Segmentation fault (core dumped)
------
Dockerfile:8
--------------------
   6 |     
   7 |     # Restore packages
   8 | >>> RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
   9 |     
  10 |     # Build project
--------------------
ERROR: failed to solve: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 139
```

As I told you in part 1 of this post, QEMU and .NET doesn't really work well together. I'm not going to spend any more time trying to fix this, because this is an issue with QEMU. I'm going to move directly to Cross Compilation.

## **Using cross-compilation**

# **Creating the AWS infrastructure**

# **Testing the app**

# **Benchmark comparing the app running on an ARM64 ECS Fargate service versus the same app running on an AMD64 ECS Fargate service**
