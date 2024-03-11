---
title: "Building and deploying a .NET 8 App on ARM64 processors with Azure Pipelines and AWS ECS Fargate. Part 1: How to build multi-platform images."
date: 2024-03-01T09:42:30+01:00
tags: ["dotnet", "containers", "azure", "aws", "devops", "arm64"]
description: "TBD"
draft: true
---

If you examine the enhancements in the last .NET versions, you'll see that each one of them brings quite a few improvements for ARM64 processors. The argument for using ARM64 processors over AMD64 is that ARM64 processors are cheaper, more efficient, and can reduce the carbon footprint.

But, how easy is it to work with them in .NET? Let's do a little test in this post. Let's check out the entire process of building and deploying a simple .NET 8 API to an ARM64 processor.

My machine has an AMD64 processor, and I want to create a container that targets an ARM64 processor. This is the most common scenario when trying to build your application on most CI providers, like Azure Pipelines, GitHub Actions, or GitLab CI. They don't offer hosted ARM64 runners.

Obviously, one option is to run your own builder instances that match the target architecture, but that's my least favorite option. You have to create your own VM instances somewhere, set them up, configure them to run with your selected CI runner, maintain them, etc, etc.    
This option might be your go-to solution, especially when you're working in a large enterprise, where dozens and dozens of builds are executed every day. Having your own CI runner gives you the ability to set it up to your specific software and hardware needs. Also, in an environment where hundreds of builds are triggered every hour, it is cheaper to use a self-hosted runner instead of the ones offered by the CI provider itself.

Obviously, I could tell you that if you want to work with ARM64 processors, you just need to create a few ARM64 machines, accommodate the Dockerfile of your .NET 8 API to target the ARM64 architecture, like the next code snippet, run a simple ``docker build -t my-api`` command, and you're done.

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

But that would be too easy, right? For the sake of this post, let's keep this option off the table. Imagine you want to **create a container image that targets an ARM64 processor using an AMD64 processor**. A very common scenario would be trying to create an image that targets an ARM64 processor using a hosted runner in Azure DevOps or GitHub Actions, because neither of them supports ARM64 hosted runners. (I know that GitHub Actions ARM64 runners are a thing nowadays, but they're still in a private beta state).

In part 1 of this post, **we will explore the current options for building a .NET 8 API container image that targets an ARM64 processor using an AMD64 processor.**.
 
# **Docker buildx command**

In order to build multi-platform container images, we need to make use of the ``docker buildx`` command. ``Buildx`` is a Docker CLI plugin that extends the docker build command with the full support of the features provided by Moby BuildKit builder toolkit.

By default, a build executed with ``buildx`` will build an image for the architecture that matches the host machine. This way, you get an image that runs on the same machine you are working on. In order to build for a different architectures, you can set the ``--platform`` flag, e.g. ``--platform=linux/arm64``.

```shell
docker buildx build --platform=linux/arm64 -t my-api .
```

When you want to create a multi-platform image you don't to specifically use the ``buildx`` syntax. ``Buildx`` is a drop-in replacement for the legacy build client used in earlier versions of Docker Engine and Docker Desktop. In newer versions of Docker Desktop and Docker Engine, **you're using Buildx by default when you invoke the docker build command**, so every time you're using the ``docker build`` command, in fact you're using the ``buildx`` command.


# **.NET images**

.NET images support multiple platforms, which means that a single image may contain variants for different architectures. When you run an image with multi-platform support, Docker automatically selects the image that matches your OS and architecture.

The .NET images provide a variety of architectures. In the following example, we can see that the .NET 8 runtime image uses a multi-platform tag image:

```bash
$ docker manifest inspect mcr.microsoft.com/dotnet/runtime:8.0 | grep architecture
            "architecture": "amd64",
            "architecture": "arm",
            "architecture": "arm64",
```
When running this image on an  ``AMD64`` machine, the ``AMD64`` variant is pulled and run, and exactly the same happens with an ``ARM64`` one.    
When you want to run the ``build`` command, you can set the ``--platform`` attribute to specify the target platform for the build output, e.g. ``linux/amd64``, ``linux/arm64``, etc.

Let me give you a practical example to see how it works. The next Dockerfile is the default one that Visual Studio creates when you create a new .NET 8 API.

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

Obviously, my machine has an AMD64 processor, so it creates an image that target ``linux/amd64``.

Now, let's specify another platform. 

```shell
$ docker build --platform=linux/arm64 -t my-api .
[+] Building 82.1s (18/18) FINISHED
$ docker inspect my-api -f "{{.Os}}/{{.Architecture}}"
linux/arm64
```

It takes considerably much more time to build the image, but we end up with an ARM64 image. The interesting part here is that we haven't changed anything at all in the Dockerfile. That's because the .NET 8 images are using a multi-platform tag image. If we don't specify the ``--platform`` attribute, it uses an image that targets the host machine architecture. If we specify the ``--platform`` attribute, then it picks the concrete image.

The .NET team also publishes architecture-specific images, such as ``mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8``. Those images are specific for a given architecture, if we're dead set on using a specific architecture and there is no need to create images for multiple architectures, then you can use them.

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

Now that we have some context about what's the ``buildx`` command used for and how the .NET multi-platform images works. Let's get down to business.  You can build multi-platform images using three different strategies:
- Building the image on a machine with the desired target architecture. In plain words, do you want to build an ARM64 container image? Then build it in an ARM64 machine.
- Using emulation.
- Using a stage in your Dockerfile to ``cross-compile`` to different architectures.


## ** 1. Emulation**

The emulation software used for building multi-platform images is named [QEMU](https://www.qemu.org/). 

If you're using Docker Desktop to create your images, then it supports it out of the box. If you're using only Docker Engine, you'll need to install it using something like [qemu-user-static](https://github.com/multiarch/qemu-user-static).

Emulation is the easiest option of the three available, because it requires no changes at all to your Dockerfile. The BuildKit automatically detects the secondary architectures that are available and when BuildKit needs to run a binary for a different architecture, it automatically loads it.

You can use the ``docker buildx ls`` command shows what emulators are installed in your machine. 

```shell
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT STATUS  BUILDKIT PLATFORMS
default *       docker
\_ default       default         running v0.12.5  linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```

Emulation with QEMU is much more slower than native builds and requires much more processing power. Moreover, QEMU doesn't quite work well with .NET. 

Remember the previous section when we built a couple of ARM64 images using the ``--platform`` image? We were using emulation to build them. A simple .NET 8 "Hello World" API was taking more than 80 seconds to build and the CPU was near 100%, now image how long will it take to build a real-world application.

A much better option than emulation is to use Cross-Compilation.

# **2. Cross-Compilation**

Using cross-compilation means leveraging the capabilities of a compiler to build for multiple platforms, without the need for emulation. The idea behind it is to use a multi-stage build and in the build stage compile your code for the target architecture, and in the run stage configure the runtime to be exported to the final image.


This approach involves using a few pre-defined build arguments that you have access to in your Docker builds: BUILDPLATFORM and TARGETPLATFORM (and derivatives, like TARGETOS). These build arguments reflect the values you pass to the --platform flag.

For example, if you invoke a build with --platform=linux/amd64, then the build arguments resolve to:

TARGETPLATFORM=linux/amd64
TARGETOS=linux
TARGETARCH=amd64

The first thing you'll need to do is use the ``BUILDPLATFORM`` argument to pin the builder to use the host native architecture as the build platform. This is to prevent emulation. That translates to adding ``--platform=$BUILDPLATFORM`` to the ``FROM`` instruction for the initial base stage.

