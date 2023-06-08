---
title: "Testing Chiseled Ubuntu container images with .NET"
date: 2023-06-05T16:43:07+02:00
tags: ["docker", "containers", "dotnet", "security", "ubuntu", "debian"]
description: "In this blog post, we will explore what're the Chiseled Ubuntu Containers and how can be used with .NET to create a more secure environment for running your containerized applications."
draft: true
---

# **What are chiseled Ubuntu containers?**

Chiselled Ubuntu images are inspired by the "distroless" concept, meaning they contain only your application and its runtime dependencies, without any additional operating system-level packages or libraries. This makes them lightweight, secure, and efficient.

These images enhances the security of the base images thanks to:

- Ultra-small image size, which reduces the potential attack surface.
- No package manager or shell installed.
- Uses a non-root user by default.

Faster starting times are one of the main performance benefits of Chiselled Ubuntu images. They can start up more quickly than traditional container images since they are significantly more lightweight and do not contain any unnecessary dependencies.

Right now Chiseled Ubuntu images are only available from Ubuntu version 22.04 (Jammy). 

This kind of images are built using Ubuntu 22.04 OS, but are heavily stripped down to include only what’s really necessary, the next image shows a comparison of a full Ubuntu image vs a Chiseled Ubuntu one.

![chiseled-ubuntu-vs-non-chiseled-ubuntu](/img/chiseled-ubuntu-vs-non-chiseled-ubuntu.png)


# **.NET in Chiseled Ubuntu Containers**

A benefit of using chiselled Ubuntu images for your containerised .NET apps is their reduced size, they are significantly smaller than traditional container images.   

In addition to not including any operating system-level packages or libraries that are not required at runtime, chiselled Ubuntu images do not include any package manager nor shell.

Comparing the size of the Ubuntu-based .NET containers using both types of images shows that the chiselled Ubuntu image is only half the size.

![chiseled-vs-standard-comparison](/img/chiseled-vs-standard-comparison.png)

Right now (06/06/2023), Chiseled Ubuntu container images are available for .NET 6, .NET 7 and .NET 8, but are **still in preview**.

The .NET 6 and .NET 7 images for AMD64 and ARM architectures can be found only in the nightly repositories while still in preview.

**.NET 6 Ubuntu Chiseled images for Amd64 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:6.0-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:6.0-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:6.0-jammy-chiseled``

**.NET 7 Ubuntu Chiseled images for Amd64 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:7.0-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:7.0-jammy-chiseled``

**.NET 6 Ubuntu Chiseled images for Arm64 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:6.0-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:6.0-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:6.0-jammy-chiseled-arm64v8``

**.NET 7 Ubuntu Chiseled images for Arm64 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:7.0-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:7.0-jammy-chiseled-arm64v8``

**.NET 6 Ubuntu Chiseled images for Arm32 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:6.0-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:6.0-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:6.0-jammy-chiseled-arm32v7``

**.NET 7 Ubuntu Chiseled images for Arm64 architecture**
- ``mcr.microsoft.com/dotnet/nightly/runtime:7.0-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/nightly/aspnet:7.0-jammy-chiseled-arm32v7``

The .NET 8 images are also still in preview (.NET 8 has not been released yet), but can be found in the official repositories.

**.NET 8 Ubuntu Chiseled images for Amd64 architecture**
- ``mcr.microsoft.com/dotnet/runtime:8.0-preview-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/runtime-deps:8.0-preview-jammy-chiseled``
- ``mcr.microsoft.com/dotnet/aspnet:8.0-preview-jammy-chiseled``

**.NET 8 Ubuntu Chiseled images for Arm64 architecture**
- ``mcr.microsoft.com/dotnet/runtime:8.0-preview-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/runtime-deps:8.0-preview-jammy-chiseled-arm64v8``
- ``mcr.microsoft.com/dotnet/aspnet:8.0-preview-jammy-chiseled-arm64v8``

**.NET 8 Ubuntu Chiseled images for Arm32 architecture**
- ``mcr.microsoft.com/dotnet/runtime:8.0-preview-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/runtime-deps:8.0-preview-jammy-chiseled-arm32v7``
- ``mcr.microsoft.com/dotnet/aspnet:8.0-preview-jammy-chiseled-arm32v7``


As you can see the Chiseled Ubuntu images are only available for the ``runtime``, ``runtime-deps`` and ``aspnet`` image, there isn't a Chiseled image with the .NET SDK because there isn't a specific need for it. Chiseled images are intended for running .NET applications, not for compiling them. For compiling the app, you can use any of the other available base images.

![chiseled-vs-standard-size-comparison](/img/chiseled-vs-standard-size-comparison.png)


# **Rootless .NET containers**

Running containers with a non-root user is a good security practice. If you run your app as ``root``, your app process can do anything in the container, like modify files, install packages, or run arbitrary executables.   
That’s a concern if your app is ever attacked. If you run your app as non-root, your app process cannot do much, greatly limiting what a bad actor could accomplish.

The .NET Chiseled Ubuntu images don't include the ``root`` user or include root-elevating commands like ``sudo`` or ``su``, which means that it is **not possible to exercise capabilities and operations that require root**.

If your app need access to privileged resources, you can add a root user within your Dockerfile. You are not prevented from that, but by default it is best to have a user with restricted permissions.

One of the pain points of using a rootless user is that the container can't use port 80, because port 80 is a privileged port that requires root permission.

The **.NET Chiseled Ubuntu images are using port 8080**, so keep that in mind when running you app.

```bash
docker run -p 5055:8080 aspnetapp
```

The next code snippet shows how the non-user root user is setup into Ubuntu Chiseled image.

```yml
RUN groupadd \
        --system \
        --gid=64198 \
        app \
    && adduser \
        --uid 64198 \
        --gid 64198 \
        --shell /bin/false \
        --system \
        app \
    && install -d -m 0755 -o 64198 -g 64198 "/rootfs/home/app" \
    && mkdir -p "/rootfs/etc" \
    && rootOrAppRegex='^\(root\|app\):' \
    && cat /etc/passwd | grep $rootOrAppRegex > "/rootfs/etc/passwd" \
    && cat /etc/group | grep $rootOrAppRegex > "/rootfs/etc/group"
```
If you want take a look by yourself, [here](https://github.com/dotnet/dotnet-docker/blob/b119f8a4c19f10be2ae0b058a52d3a8f23685e90/src/runtime-deps/8.0/jammy-chiseled/amd64/Dockerfile) is the Microsoft Github repository where the Dockerfile is being defined.


> The use of a non-root user is not unique to .NET Chiseled Ubuntu containers anymore.   
> Starting from .NET 8, all images, regardless of the distro, will transition to using a non-root user.


# **Comparison between Chiseled Ubuntu containers and Debian 12 containers**

The default operating system for working with containers in .NET has always been Debian. When we create a new Dockerfile for our app from Visual Studio, it always generates a Dockerfile that uses Debian as the base image, regardless of whether the app is built with .NET Core 3.1, .NET 5, .NET 7, or .NET 8.

Therefore, I believe that making a comparison between a "Hello World" app built with .NET 7 using Debian 11 (Bullseye) as the base image and another one using Ubuntu Chiseled (Jammy) as the base image could be interesting.

## **Image size comparison**

### **Base image size comparison**

First, a comparison between the size of the .NET 7 base images with Debian 11 as a base image or with a Chiseled Ubuntu 22.04 base image.

| Base Image                        | Operating System      | Image size |
|-----------------------------------|-----------------------|------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             | 118MB      |
| runtime-deps:7.0-jammy-chiseled   | Chiseled Ubuntu 22.04 | 13MB       |
| runtime:7.0-bullseye-slim         | Debian 11             | 191MB      |
| runtime:7.0-jammy-chiseled        | Chiseled Ubuntu 22.04 | 86.2MB     |
| aspnet:7.0-bullseye-slim          | Debian 11             | 213MB      |
| aspnet:7.0-jammy-chiseled         | Chiseled Ubuntu 22.04 | 108MB      |


The difference in size for the base images are quite considerable. When building an app, I personally tend to favour the use of the ``runtime-deps`` image over the other two, and as we can see in the comparison, the space savings achieved by using chiseled containers are significant.

Now, let's make another comparison adding a .NET app on top of the base image.

### **API image size comparison**

The API is nothing fancy, it is the default "WeatherForecast" minimal Api. 

It makes no sense to use the ``runtime`` base image for an API, so we're going to make a comparison using only the ``runtime-deps`` and the ``aspnet`` base images.

| Base Image                        | Operating System      | Image size |
|-----------------------------------|-----------------------|------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             | 219MB      |
| runtime-deps:7.0-jammy-chiseled   | Chiseled Ubuntu 22.04 | 115MB      |
| aspnet:7.0-bullseye-slim          | Debian 11             | 217MB      |
| aspnet:7.0-jammy-chiseled         | Chiseled Ubuntu 22.04 | 112MB      |


The difference in size after adding a .NET 7 API keeps being quite considerable between using Debian 11 or a Chiseled Ubuntu image.   
For applications that are deployed at scale and much more so in distributed and edge computing environments, using the Chiseled Ubuntu as a base image over Debian 11 can result in significant storage and deployment cost savings.

## **Resource usage comparison**

- The app used for this comparison has the following features:
    - It is a simple "Hello World" API built with .NET 7.
    - Uses the ``runtime-deps`` base image.

- The measurements are taken with [Microsoft Crank](https://github.com/dotnet/crank)
    - Crank is the benchmarking infrastructure used by the .NET team to run benchmarks including scenarios from the TechEmpower Web Framework Benchmarks.

TODO

# **Security issues**

Chiselled Ubuntu images are designed with security in mind. Their reduced size greatly diminishes the attack surface of the images, making them less likely to be affected by vulnerabilities.

Also keep in mind that they do not include a package manager or shell, which completely disarms certain classes of attacks, but that also means that you won't be able to login into a running container and start executing shell commands like you can do with regular images.   

Running scripts or commands in a running container is not considered a good security practice. However, if you are currently doing it, please be aware that by default, you won't be able to do so with this type of image.

## **Vulnerability scanning comparison**

In this section we'll compare the security vulnerabilities between an app using Debian 11 (Bookworm) as the base image and another one using Chiseled Ubuntu 22.04 (Jammy) as the base image. 

Nowadays there are numerous container security scanners available (in one of my previous [posts](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/) I discussed some of the well-known options and how to integrate them with Azure Pipelines). For this comparison, we will be using:

- [Clair via AWS ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)
- [Trivy](https://github.com/aquasecurity/trivy)

Why test it using two security scanners instead of a single one? More data is always useful.

- **Clair via AWS ECR** 

| Base Image                        | Operating System      | Security issues summary                              |
|-----------------------------------|-----------------------|------------------------------------------------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             |  3 Medium, 5 Low, 33 Informational, 7 Undefined      |
| runtime-deps:7.0-jammy-chiseled   | Chiseled Ubuntu 22.04 |  None                                                |
| aspnet:7.0-bullseye-slim          | Debian 11             |  3 Medium, 5 Low, 33 Informational, 7 Undefined      |
| aspnet:7.0-jammy-chiseled         | Chiseled Ubuntu 22.04 |  None                                                |

- **Trivy**

| Base Image                        | Operating System      | Security issues summary                              |
|-----------------------------------|-----------------------|------------------------------------------------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             |  LOW: 61, MEDIUM: 4, HIGH: 17, CRITICAL: 1           |
| runtime-deps:7.0-jammy-chiseled   | Chiseled Ubuntu 22.04 |  LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0             |
| aspnet:7.0-bullseye-slim          | Debian 11             |  LOW: 61, MEDIUM: 4, HIGH: 17, CRITICAL: 1           |
| aspnet:7.0-jammy-chiseled         | Chiseled Ubuntu 22.04 |  LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0             |

It is well-known that Debian distros have a number of security issues. As new versions are released, some problems get resolved while new ones emerge. Simply updating to a newer version of Debian doesn't guarantee the disappearance of security issues.

From this comparison, we can see that the containers that are using a Chiseled Ubuntu image as the base image have no security problems, which is quite surprising.   

Until now, Alpine has been considered the most secure distro for .NET containers. However, we have observed that Chiseled Ubuntu images provide the same level of security while retaining the advantages of being an Ubuntu distro.


# **Migrate a containerized app to Chiseled Ubuntu containers**

In this section, I will migrate a containerized WebAPI that is currently using Debian 11 as the base image, more specifically the ``runtime-deps:7.0-bullseye-slim` image, to  use a Chiseled Ubuntu image as its base image.

## **Application**

The app can be found here:
- https://github.com/karlospn/opentelemetry-metrics-demo

It is a bookStore API built using .NET 7. It allows us to do the following actions:
- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and delete orders.

## **Migration process**

1.  Change the base image on the Dockerfile to start using a ubuntu chiseled one.

This application does not perform any operation that requires a user with root permissions. However, if for any reason we needed a root user, we can simply create it in the Dockerfile using the command "RUN adduser".

```yaml
FROM mcr.microsoft.com/dotnet/sdk:7.0-bullseye-slim AS build-env
WORKDIR /app

# Copy everything
COPY . ./

# Restore packages
RUN dotnet restore -s "https://api.nuget.org/v3/index.json" \
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
	--runtime linux-x64

# Build runtime image
FROM mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Set Entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```

2. In the ``docker-compose`` file we have to use port 8080 instead of port 80

```yaml
  app:
    build:
      context: ./
      dockerfile: ./src/BookStore.WebApi/Dockerfile
    depends_on:
      - mssql
      - otel-collector
    ports:
      - 5001:8080
    environment:
      ConnectionStrings__DbConnection: Server=mssql;Database=BookStore;User Id=SA;Password=P@ssw0rd?;Encrypt=False
      Otlp__Endpoint: http://otel-collector:4317
    networks:
      - metrics
```

And we're done! Really easy.

# **Conclusion**

