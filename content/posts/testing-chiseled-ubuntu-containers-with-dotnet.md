---
title: "Testing Chiseled Ubuntu container images with .NET"
date: 2023-06-05T16:43:07+02:00
tags: ["docker", "containers", "dotnet", "security", "ubuntu", "debian"]
description: "The objective of this post is to test Ubuntu Chiseled images in combination with .NET, evaluating whether it is truly worthwhile to migrate our .NET apps to utilize these base images or not."
draft: true
---

# **What are Chiseled Ubuntu images?**

Chiseled Ubuntu images are inspired by the "distroless" concept, meaning they contain only your application and its runtime dependencies, without any additional operating system-level packages or libraries. This makes them lightweight, secure, and efficient.

Chiseled Ubuntu images will enhance the security of your containers thanks to:

- Ultra-small size, which reduces the potential attack surface.
- No package manager or shell installed.
- Uses a non-root user by default.

Faster starting times are one of the main performance benefits of Chiseled Ubuntu images. They can start up more quickly than traditional container images since they are significantly more lightweight and don't contain any unnecessary dependencies.

Right now, Chiseled Ubuntu images are only available from Ubuntu version 22.04 (Jammy distro). 

This kind of images are built using Ubuntu 22.04 OS, but are heavily stripped down to include only what’s really necessary, the next image shows a comparison of a full Ubuntu image vs a Chiseled Ubuntu one.

![chiseled-ubuntu-vs-non-chiseled-ubuntu](/img/chiseled-ubuntu-vs-non-chiseled-ubuntu.png)


# **.NET and Chiseled Ubuntu images**

A benefit of using Chiseled Ubuntu images for your containerised .NET apps is their reduced size, they are significantly smaller than traditional container images.   

In addition to not including any operating system-level packages or libraries that are not required at runtime, they do not include any package manager nor shell.

Comparing the size of the Ubuntu-based .NET image using both types of images shows that the Chiseled Ubuntu image is approximately half the size.

![chiseled-vs-standard-comparison](/img/chiseled-vs-standard-comparison.png)

Right now (06/06/2023), Chiseled Ubuntu images are available for .NET 6, .NET 7 and .NET 8, but are **still in preview**.

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

**.NET 7 Ubuntu Chiseled images for Arm32 architecture**
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


As you can see the Chiseled Ubuntu images are only available for the ``runtime``, ``runtime-deps`` and ``aspnet`` image, there isn't a Chiseled image with the .NET SDK because there isn't a specific need for it.    
Chiseled images are intended for running .NET applications, not for compiling them. For compiling the app, you can use any of the other available base images.

![chiseled-vs-standard-size-comparison](/img/chiseled-vs-standard-size-comparison.png)


# **Rootless .NET containers**

Running containers with a non-root user is a good security practice. If you run your app as ``root``, your app process can do anything in the container, like modify files, install packages or run arbitrary executables.   
That’s a concern if your app is ever attacked. If you run your app as non-root, your app process cannot do much, greatly limiting what a bad actor could accomplish.

The .NET Chiseled Ubuntu images don't include the ``root`` user or include root-elevating commands like ``sudo`` or ``su``, which means that it is **not possible to exercise capabilities and operations that requires root permissions**.

If your app needs access to privileged resources, you can add a root user within your Dockerfile, you are not prevented from that, but by default it is best to have a user with restricted permissions.

One of the pain points of running the app with a rootless user is that it can't use port 80, because port 80 is a privileged port that requires root permissions.

By default, **.NET Chiseled Ubuntu images are using port 8080**, so keep that in mind when running you app.

```bash
docker run -p 5055:8080 aspnetapp
```

The next code snippet shows how the non-user root user is setup in the Ubuntu Chiseled base image.

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
If you want take a deeper look, [here](https://github.com/dotnet/dotnet-docker/blob/b119f8a4c19f10be2ae0b058a52d3a8f23685e90/src/runtime-deps/8.0/jammy-chiseled/amd64/Dockerfile) is the Microsoft Github repository where the Ubuntu Chiseled base image Dockerfile lives.


> The use of a non-root user is not unique to .NET Chiseled Ubuntu containers anymore.   
> Starting from .NET 8, all images, regardless of the distro, will transition to using a non-root user.


# **Chiseled Ubuntu containers vs Debian containers**

The default operating system for working with containers in .NET has always been Debian. When we create a new Dockerfile for our app from Visual Studio, it always generates a Dockerfile that uses Debian as base image, regardless of whether the app is built with .NET Core 3.1, .NET 5, .NET 7, or .NET 8.

Therefore, I believe that making a comparison between an app built with .NET 7 using Debian (Bullseye distro) as the base image and another one using Ubuntu Chiseled (Jammy distro) as the base image could be interesting.

## **Image size benchmark**

### **Base image size comparison**

First of all, let's compare the size of the base images without any .NET app on top, let's take a look at the base images that can be found on the Microsoft Container Registry.

| Base Image                        | Operating System      | Image size |
|-----------------------------------|-----------------------|------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             | 118MB      |
| runtime-deps:7.0-jammy-chiseled   | Chiseled Ubuntu 22.04 | 13MB       |
| runtime:7.0-bullseye-slim         | Debian 11             | 191MB      |
| runtime:7.0-jammy-chiseled        | Chiseled Ubuntu 22.04 | 86.2MB     |
| aspnet:7.0-bullseye-slim          | Debian 11             | 213MB      |
| aspnet:7.0-jammy-chiseled         | Chiseled Ubuntu 22.04 | 108MB      |


The difference in size for the base images are quite considerable.    
When containerizing an app, I personally tend to favour the use of the ``runtime-deps`` image over the other two (``runtime`` and ``aspnet``) images, but as we can see in the above table, the space savings achieved by using a chiseled image is significant on all 3 base images.

Now, let's make another comparison, but adding a .NET 7 app on top of the base image.

### **API image size comparison**

For this comparison we have created an image with a .NET 7 API using different base images.   
The API is nothing fancy, it is the default "WeatherForecast" minimal Api. 

It makes no sense to use the ``runtime`` base image for an API, so we're going to do a comparison using only the ``runtime-deps`` and the ``aspnet`` base images.

| Image                                                             | Operating System      | Image size |
|-------------------------------------------------------------------|-----------------------|------------|
| .NET 7 API using ``runtime-deps:7.0-bullseye-slim`` as base image     | Debian 11             | 219MB      |
| .NET 7 API using ``runtime-deps:7.0-jammy-chiseled`` as base image    | Chiseled Ubuntu 22.04 | 115MB      |
| .NET 7 API using ``aspnet:7.0-bullseye-slim`` as base image           | Debian 11             | 217MB      |
| .NET 7 API using ``aspnet:7.0-jammy-chiseled`` as base image          | Chiseled Ubuntu 22.04 | 112MB      |


Adding a .NET 7 app on top of the base images increases the size of the resulting images on roughly 100Mb, but the difference in size between using Debian or a Chiseled Ubuntu image remains quite considerable.   

For applications that are deployed at scale and much more so in distributed and edge computing environments, using a Chiseled Ubuntu image as base image over Debian can result in significant storage and deployment cost savings.

## **Resource usage benchmark**

The app used for this benchmark is the default .NET 7 "WeatherForecast" minimal Api

To generate load, we'll be using [Bombardier](https://github.com/codesenberg/bombardier).

And we'll be running 2 scenarios.

**Scenario 1**
  - 1200 total requests
  - 60 seconds duration
  - 20 requests/sec
  - bombardier command: ``bombardier --connections=20 --requests=1200 --duration=60s --rate=20 --fasthttp http://localhost:5051/weatherforecast``

**Scenario 2**
  - 3000 total requests
  - 60 seconds duration
  - 50 requests/sec
  - bombardier command: ``bombardier --connections=50 --requests=3000 --duration=60s --rate=50 --fasthttp http://localhost:5051/weatherforecast``   


Each scenario will be executed 20 times with an app that uses the ``runtime-deps:7.0-jammy-chiseled`` as base image, and another 20 times with the same app but this time it will be using the ``runtime-deps:7.0-bullseye-slim`` as base image.   

Once we have all the execution times, we will calculate the average of the **MAX CPU USAGE** and **MAX MEMORY USAGE** for each base image.

### **Scenario 1 results**

| Image                                                          | Operation System      | Max. CPU percentage | Max. memory usage |
|----------------------------------------------------------------|-----------------------|---------------------|-------------------|
| .NET 7 API using ``runtime-deps:7.0-bullseye-slim`` as base image  | Debian 11             | 0.04%               | 30.14MB           |
| .NET 7 API using ``runtime-deps:7.0-jammy-chiseled`` as base image | Chiseled Ubuntu 22.04 | 0.03%               | 27.56MB           |

### **Scenario 2 results**

| Image                                                          | Operation System      | Max. CPU percentage | Max. memory usage |
|----------------------------------------------------------------|-----------------------|---------------------|-------------------|
| .NET 7 API using ``runtime-deps:7.0-bullseye-slim`` as base image  | Debian 11             | 0.06%               | 32.21MB           |
| .NET 7 API using ``runtime-deps:7.0-jammy-chiseled`` as base image | Chiseled Ubuntu 22.04 | 0.06%               | 34.48MB           |


As we can see, CPU usage is practically identical whether we use one base image or the other.    
However, there is a slight difference in memory usage, the version of the API that uses the Chiseled Ubuntu image as base image has consumed less memory in each and every test conducted.


# **Security issues**

Chiseled Ubuntu images are designed with security in mind. Their reduced size greatly diminishes the attack surface of the images, making them less likely to be affected by vulnerabilities.

Also keep in mind that they do not include a package manager or shell, which completely disarms certain classes of attacks, but that also means that you won't be able to login into a running container and start executing shell commands like you can do with regular images.   

Running scripts or commands on-demand on a running container is not considered a good security practice. However, if you are currently doing it, please be aware that by default, you won't be able to do so with this type of image.

## **Vulnerability scanning benchmark**

**In this section we'll compare the number of security vulnerabilities between a .NET 7 API using Debian (Bullseye distro) as base image and the same .NET 7 API using Chiseled Ubuntu (Jammy distro) as base image.** 

Nowadays there are numerous container security scanners available (in one of my previous [posts](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/), I discussed some of the most well-known options and how to integrate them with Azure Pipelines). For this comparison, we will be using:

- [Clair via AWS ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)
- [Trivy](https://github.com/aquasecurity/trivy)

Why test it using two security scanners instead of a single one? More data is always useful.

- **Scan results using Clair via AWS ECR** 

| Image                                                       | Operating System      | Security issues summary                              |
|------------------------------------------------------------------|-----------------------|------------------------------------------------------|
| .NET 7 API using ``runtime-deps:7.0-bullseye-slim`` as base image    | Debian 11             |  3 Medium, 5 Low, 33 Informational, 7 Undefined      |
| .NET 7 API using ``runtime-deps:7.0-jammy-chiseled`` as base image   | Chiseled Ubuntu 22.04 |  None                                                |
| .NET 7 API using ``aspnet:7.0-bullseye-slim`` as base image          | Debian 11             |  3 Medium, 5 Low, 33 Informational, 7 Undefined      |
| .NET 7 API using ``aspnet:7.0-jammy-chiseled`` as base image         | Chiseled Ubuntu 22.04 |  None                                                |

- **Scan results using Trivy**

| Image                                                            | Operating System      | Security issues summary                              |
|------------------------------------------------------------------|-----------------------|------------------------------------------------------|
| .NET 7 API using ``runtime-deps:7.0-bullseye-slim`` as base image    | Debian 11             |  LOW: 61, MEDIUM: 4, HIGH: 17, CRITICAL: 1           |
| .NET 7 API using ``runtime-deps:7.0-jammy-chiseled`` as base image   | Chiseled Ubuntu 22.04 |  LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0             |
| .NET 7 API using ``aspnet:7.0-bullseye-slim`` as base image          | Debian 11             |  LOW: 61, MEDIUM: 4, HIGH: 17, CRITICAL: 1           |
| .NET 7 API using ``aspnet:7.0-jammy-chiseled`` as base image         | Chiseled Ubuntu 22.04 |  LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0             |


It is well-known that when we perform a security scan on an image using a Debian distro, a good handful of vulnerabilities appear. This is nothing new; it occurred in older versions of Debian and continues to occur in newer versions of Debian. Simply updating to a newer version of Debian doesn't guarantee the disappearance of security issues.

From this comparison, we can see that the container that is using a Chiseled Ubuntu image as base image has 0 security issues, which is quite surprising.   

Until now, Alpine has been considered the most secure distro for .NET containers. However, it seems that Chiseled Ubuntu images can provide the same level of security while retaining the advantages of being an Ubuntu distro.


# **Migrate a containerized app to start using a Chiseled Ubuntu image as its base image**

In this section, I will migrate a containerized API that is currently using Debian 11 as the base image, more specifically the ``runtime-deps:7.0-bullseye-slim`` image, to start using a Chiseled Ubuntu image as its base image.

## **Application**

The app can be found here:
- https://github.com/karlospn/opentelemetry-metrics-demo

It is a bookStore API built using .NET 7. It allows us to do the following actions:
- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and delete orders.

## **Migration process**

1.  Change the base image on the Dockerfile to ``mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled``

This application does not perform any operation that requires a user with ``root`` permissions. However, if for any reason we needed a root user, we can create it in the Dockerfile using the ``RUN adduser`` command.

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

2. The app now starts on port ``8080`` instead of port ``80``, so change it in the ``docker-compose`` file.

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

And done!

# **Conclusion**

In this post, we have seen what Chiseled Ubuntu images are, what characteristics they have, and how to use them with .NET.

The most important point of this post was to assess the viability of Chiseled Ubuntu images and whether we can use them as the base image in our .NET applications.    
For that purpose, we conducted a series of benchmarks and comparisons between a .NET 7 API using a Debian base image and the same .NET 7 API using a Chiseled Ubuntu base image.

- Why did we choose to compare it to Debian?    

Because Debian is the recommended distribution by Microsoft since .NET Core 2.1. In fact, if we try to add a Dockerfile to a .NET app using Visual Studio right now, the generated Dockerfile will contain a Debian-based base image.

- And what has been the result of this comparison?    

Well, the truth is that the results are quite surprising. Using a Chiseled Ubuntu image as base image of our .NET 7 app results in:
- Smaller image size.
- Better memory usage.
- Fewer security vulnerabilities.

As of today (06/09/2023), Chiseled Ubuntu images are still in preview. However, once they become generally available (GA), there is no reason not to use them over other distributions.