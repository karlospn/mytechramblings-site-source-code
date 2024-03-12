---
title: "Building and deploying a .NET 8 App on an ARM64 processor using Azure Pipelines and AWS ECS Fargate. Part 1: How to build multi-platform images"
date: 2024-03-01T09:42:30+01:00
tags: ["dotnet", "containers", "azure", "aws", "devops", "arm64"]
description: "In this two-part series, I’m going to show you how to build and deploy a .NET 8 app container image that targets an ARM64 processor. In part 1, I’ll be discussing some key concepts that you should know about how to build multi-platform images with .NET."
draft: true
---

> This is a two-part series post.
> - **Part 1**: Key concepts about how to build multi-platform images.
> - **Part 2**: A practical example of how to build a container image targeting an ARM64 processor using Azure Pipelines and how to deploy it on AWS ECS Fargate.    
It will also include a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container.


If you examine the enhancements in the latest .NET versions, you'll notice that each one of them brings quite a few improvements for ARM64 processors. The argument for using ARM64 processors over AMD64 is that ARM64 processors are cheaper, more efficient, and can reduce the carbon footprint.

But, how easy is it to work with ARM64 in .NET? Let's conduct a little test in this post.    
Let's explore the entire process of building and deploying a simple .NET 8 API to an ARM64 host.

My machine has an AMD64 processor, and I want to create a container image that targets an ARM64 processor. This is the most common scenario when attempting to build your application on most CI providers, like Azure Pipelines, GitHub Actions, or GitLab CI. They don't offer hosted ARM64 runners.

Obviously, one option is to run your own builder instances that match the target architecture, but that's my least favorite option. You have to create your own VM instances somewhere, set them up, configure them to run with your selected CI runner, maintain them, etc, etc.    

This option might be your go-to solution, especially when you're working in a large enterprise, where dozens and dozens of builds are executed every day. Having your own CI runner gives you the ability to set it up according to your specific software and hardware needs. Also, in an environment where hundreds of builds are triggered every hour, it is cheaper to use a self-hosted runner instead of the ones offered by the CI provider itself.

Obviously, I could tell you that if you want to build an image targeting an ARM64 processor, you just need to create an ARM64 machine, accommodate the Dockerfile of your .NET 8 API to target the ARM64 architecture, as shown in the next code snippet, execute a simple ``docker build`` command in the newly created machine, and you're done.

```yml
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy-arm64v8 AS build

WORKDIR /app
COPY . ./

RUN dotnet restore "./Arm64Testing.WebApi.csproj" \
    --runtime linux-arm64

RUN dotnet build "./Arm64Testing.WebApi.csproj" \
    -c Release \
    --runtime linux-arm64 \
    --no-restore

RUN dotnet publish "./Arm64Testing.WebApi.csproj" \
    -c Release \
    -o /app/publish \
    --runtime linux-arm64 \
    --no-restore \
    --no-build

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8
EXPOSE 8080
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

But that would be too easy, right? For the sake of this post, let's keep this option off the table.   

Imagine you want to **create a container image that targets an ARM64 processor using an AMD64 processor**. A very common scenario for this use case would be attempting to create a container image using a hosted runner in Azure DevOps or GitHub Actions, because neither of them supports ARM64 hosted runners.

I know that GitHub Actions ARM64 runners are a thing nowadays, but they're still in a private beta state.

In part 1 of this post, **we will explore the current options for building a .NET 8 API container image that targets an ARM64 processor using an AMD64 processor**.
 
# **docker buildx command**

In order to build multi-platform container images, we need to make use of the ``docker buildx`` command. ``Buildx`` is a Docker CLI plugin that extends the docker build command with the full support of the features provided by Moby BuildKit builder toolkit.

By default, a build executed with ``buildx`` will build an image for the architecture that matches the host machine. This way, you get an image that runs on the same machine you are working on. In order to build it for a different architecture, you need to the ``--platform`` flag, e.g. ``--platform=linux/arm64``.

```shell
docker buildx build --platform=linux/arm64 -t my-api .
```

When you want to create a multi-platform image you don't have to specifically use the ``buildx`` syntax. ``Buildx`` is a drop-in replacement for the legacy build client used in earlier versions of Docker Engine and Docker Desktop.   

In newer versions of Docker Desktop and Docker Engine, **you're using buildx by default when you invoke the docker build command**, so every time you're using the ``docker build`` command, in fact you're using the ``buildx`` command.


# **.NET images**

.NET 8 images support multiple platforms, which means that a single image may contain variants for different architectures.    
When you run an image with multi-platform support, Docker automatically selects the image that matches your OS and architecture.

The following code snippet demonstrates how the .NET 8 SDK and runtime images contain a multi-platform tag image.

```bash
$ docker manifest inspect mcr.microsoft.com/dotnet/sdk:8.0 | grep architecture
            "architecture": "amd64",
            "architecture": "arm",
            "architecture": "arm64",
```

```bash
$ docker manifest inspect mcr.microsoft.com/dotnet/runtime:8.0 | grep architecture
            "architecture": "amd64",
            "architecture": "arm",
            "architecture": "arm64",
```

```bash
$ docker manifest inspect mcr.microsoft.com/dotnet/runtime-deps:8.0 | grep architecture
            "architecture": "amd64",
            "architecture": "arm",
            "architecture": "arm64",
```
When running any of these images on an  ``AMD64`` processor, the ``AMD64`` variant will be pulled and run, and exactly the same happens with ``ARM64``.    

When you want to run the ``build`` command with a specific architecture that is not your host machine architecture, you can use the ``--platform`` attribute, e.g. ``linux/amd64``, ``linux/arm64``, etc.

Let me provide you with a practical example to demonstrate how it works. The following Dockerfile is the default one that Visual Studio creates when you create a new .NET 8 API.

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

Let's begin by simply building the image.

```shell
$ docker build -t my-api .
Building 16.8s (18/18) FINISHED
$ docker inspect my-api -f "{{.Os}}/{{.Architecture}}"
linux/amd64
```

Obviously, my machine has an AMD64 processor, so if we inspect it, we can see that the image targets a ``linux/amd64`` architecture.

Now, let's specify another platform using the ``--platform`` attribute. 

```shell
$ docker build --platform=linux/arm64 -t my-api .
[+] Building 82.1s (18/18) FINISHED
$ docker inspect my-api -f "{{.Os}}/{{.Architecture}}"
linux/arm64
```

It takes considerably more time to build the image, but we end up with an ARM64 image.  

The interesting part here is that we haven't changed anything at all in the Dockerfile. That's because the .NET 8 images are using a multi-platform tag image. If we don't specify the ``--platform`` attribute, it uses an image that targets the host machine architecture. If we specify the ``--platform`` attribute, then it uses the image that targets the architecture we have specified.

The .NET team also publishes architecture-specific images, such as ``mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8``. Those images are specific to a given architecture. If we're determined to use a specific architecture and there is no need to create images for multiple architectures, then you can use them.

Let's modify the above Dockerfile to use the ``arm64`` architecture-specific images.

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy-arm64v8 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

And try to build it.

```shell
$ docker build -t my-api .
Building 86.8s (18/18) FINISHED
$ docker inspect my-api -f "{{.Os}}/{{.Architecture}}"
linux/arm64
```
As you can see, an arm64 image also gets created. With this approach using the ``--platform`` attribute makes no sense, because we're specifically building an image that targets an ARM64 architecture. 
 

# **Multi-platform image strategies**

Now that we have some context about what the ``buildx`` command is used for and what the .NET multi-platform images are, let's get down to business. You can build multi-platform images using three different strategies:
- Building the image on a machine with the desired target architecture. In simple terms, if you want to build an ARM64 container image, then build it on an ARM64 machine.
- Using emulation.
- Using a stage in your Dockerfile to ``cross-compile`` to different architectures.


## **1. Emulation**

The emulation software used for building multi-platform images is named [QEMU](https://www.qemu.org/). 

If you're using Docker Desktop to create your images, then it supports QEMU out of the box. If you're using only Docker Engine, you'll need to install it using something like [qemu-user-static](https://github.com/multiarch/qemu-user-static).

Emulation is the easiest option of the three available, because it requires no changes at all to your Dockerfile. The BuildKit automatically detects the secondary architectures that are available and when BuildKit needs to run a binary for a different architecture, it automatically loads it.

You can use the ``docker buildx ls`` command to see what emulators are installed on your machine. 

```shell
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT STATUS  BUILDKIT PLATFORMS
default *       docker
\_ default       default         running v0.12.5  linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```

If you see different platforms listed (like linux/arm64, linux/riscv64, etc.), and you're building an image for a platform that's different from your host platform, then QEMU is being used to emulate the target platform.

Emulation with QEMU is much more slower than native builds and requires much more processing power. Moreover, QEMU doesn't quite work well with .NET. 

Remember the previous section when we built a couple of ARM64 images using the ``--platform`` attribute? We were using emulation to build them.    

On my machine, bulding a container with QEMU that contained a simple .NET 8 "Hello World" API was taking more than 80 seconds, and the CPU was peaking near 90%. Now image how long will it take to build real-world applications using emulation  (if they work at all, because as I said before, QEMU and .NET don't quite gel together).

A much better option for building .NET apps is to use cross-compilation instead of emulation.

# **2. Cross-Compilation**

Emulation is only used when building multi-architecture images or running containers for a different architecture than your host machine. If you're building an image for the same architecture as your host machine, Docker will just use the native OS and CPU architecture, and QEMU won't be involved.

That's where cross-compilation comes to play. Using cross-compilation means leveraging the capabilities of a compiler to build for multiple platforms, without the need for emulation. The idea behind it is to use a multi-stage build and in the build stage compile your code for the target architecture, and in the run stage configure the runtime to be exported to the final image.

This approach involves using a few pre-defined build arguments that you have access to in your Docker builds. For .NET we're going to use the ``BUILDPLATFORM`` and the ``TARGETARCH`` arguments. These build arguments reflect the values you pass to the ``--platform`` flag.

For example, if you invoke ``docker build`` with ``--platform=linux/arm64``, then the build arguments resolve to:

- BUILDPLATFORM=``linux/amd64`` (my host machine architecture)
- TARGETARCH=``arm64``

Let's modify the Dockerfile from the previous section to use cross-compilation. This is how the Dockerfile from the previous section looked:

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

The first thing you'll need to do is use the ``BUILDPLATFORM`` argument to pin the builder to use the host native architecture as the build platform. This is only to prevent emulation. That translates to adding ``--platform=$BUILDPLATFORM`` to the ``FROM`` instruction for the initial build stage.

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM --platform=$BUILDPLATFORM  mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM  --platform=$BUILDPLATFORM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

Next, add the ``ARG TARGETARCH`` instructions for the build stage, making the ``TARGETARCH`` build arguments available to the commands in this stage.

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM --platform=$BUILDPLATFORM  mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
ARG TARGETARCH
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj" 
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build 

FROM  --platform=$BUILDPLATFORM build AS publish
ARG BUILD_CONFIGURATION=Release
ARG TARGETARCH
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false 

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

Finally modify the ``dotnet restore``, ``dotnet build`` and ``dotnet publish`` commands so they create the application binaries for the target architecture (in our case ``arm64``).    
To achieve that, we're going to use the ``--arch`` attribute. This attribute can be used in all three commands (``dotnet restore``, ``dotnet build`` and ``dotnet publish``) and it is used to set the Runtime Identifier (RID).

```yml
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM --platform=$BUILDPLATFORM  mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
ARG TARGETARCH
WORKDIR /src
COPY ["Arm64Testing.WebApi.csproj", "."]
RUN dotnet restore "./././Arm64Testing.WebApi.csproj" --arch $TARGETARCH
COPY . .
WORKDIR "/src/."
RUN dotnet build "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build --arch $TARGETARCH

FROM  --platform=$BUILDPLATFORM build AS publish
ARG BUILD_CONFIGURATION=Release
ARG TARGETARCH
RUN dotnet publish "./Arm64Testing.WebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false --arch $TARGETARCH

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```
And we're done, let's just give it a try.

```shell
$ docker build --platform=linux/arm64 -t my-api .
[+] Building 17.7s (18/18) FINISHED
$ docker inspect my-api -f "{{.Os}}/{{.Architecture}}"
linux/arm64
```

Using cross-compilation, we've reduced the build time of a simple .NET 8 API container image that targets an ARM64 processor on my AMD64 machine from over 80 seconds to 17 seconds, and the CPU usage from nearly 90% to no more than 20%.

However, it's worth noting that while this "Hello World" .NET 8 API worked well with QEMU, more complex applications may encounter some errors when using emulation.

In part 2 of this blog post, we will attempt to perform an end-to-end process. We will build a .NET 8 API, containerize the app targeting an ARM64 processor using Azure Pipelines, and finally deploy it on AWS ECS Fargate.    
Additionally, we will conduct a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container.