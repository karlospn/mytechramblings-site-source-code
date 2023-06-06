---
title: "Chiseled Ubuntu container images and .NET"
date: 2023-06-05T16:43:07+02:00
tags: ["docker", "containers", "dotnet", "security", "ubuntu", "debian"]
description: "TBD"
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

A huge benefit of using chiselled Ubuntu images for your containerised .NET apps is their reduced size, they are significantly smaller than traditional container images.   

In addition to not including any operating system-level packages or libraries that are not required at runtime, chiselled Ubuntu images do not include any package manager nor shell.

Comparing the size of the Ubuntu-based .NET containers using both types of images shows that the chiselled Ubuntu image is only half the size.

![chiseled-vs-standard-comparison](/img/chiseled-vs-standard-comparison.png)

Right now (06/06/2023), Chiseled Ubuntu container images are available for .NET 6, .NET 7 and .NET 8, but are **still in preview**.

The .NET 6 and .NET 7 images can be found only in the nightly repositories while still in preview.

- ``runtime`` images
- ``runtime-deps`` images
- ``aspnet

<add-links>

The .NET 8 images are also still in preview (.NET 8 has not been released yet), but can be found in the official repositories.

<add-links>

As you can see the Chiseled Ubuntu images are only available for the runtime and aspnet image, there isn't a Chiseled image with the .NET SDK because there isn't a specific need for it. Chiseled images are intended for running .NET applications, not for compiling them. For compiling the app, you can use any of the other available base images.

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
```yaml
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
If you want take a look by yourself, [here]((https://github.com/dotnet/dotnet-docker/blob/b119f8a4c19f10be2ae0b058a52d3a8f23685e90/src/runtime-deps/8.0/jammy-chiseled/amd64/Dockerfile)) is the Microsoft Github repository where the Dockerfile is being defined.


> The use of a non-root user is not unique to .NET Chiseled Ubuntu containers anymore.   
> Starting from .NET 8, all images, regardless of the distro, will transition to using a non-root user.


# **Comparison between Chiseled Ubuntu containers and Debian 12 containers**

The default operating system for working with containers in .NET has always been Debian. When we create a new Dockerfile for our app from Visual Studio, it always generates a Dockerfile that uses Debian as the base image, regardless of whether the app is built with .NET Core 3.1, .NET 5, .NET 7, or .NET 8.

Therefore, I believe that making a comparison between a "Hello World" app built with .NET 7 using Debian 11 (Bullseye) as the base image and another one using Ubuntu Chiseled (Jammy) as the base image could be interesting.

## **Size comparison**

- The app is a simple "Hello World" API built with .NET 7.

| Base Image                        | Operating System      | Image size |
|-----------------------------------|-----------------------|------------|
| runtime-deps:7.0-bullseye-slim    | Debian 11             | XXXMB      |
| runtime-deps:7.0.5-jammy          | Ubuntu 22.04          | YYYMB      |
| runtime-deps:7.0.5-jammy-chiseled | Chiseled Ubuntu 22.04 | ZZZMB      |
| runtime:7.0-bullseye-slim         | Debian 11             | XXXMB      |
| runtime:7.0.5-jammy               | Ubuntu 22.04          | YYYMB      |
| runtime:7.0.5-jammy-chiseled      | Chiseled Ubuntu 22.04 | ZZZMB      |
| aspnet:7.0-bullseye-slim          | Debian 11             | XXXMB      |
| aspnet:7.0.5-jammy                | Ubuntu 22.04          | YYYMB      |
| aspnet:7.0.5-jammy-chiseled       | Chiseled Ubuntu 22.04 | ZZZMB      |


## **Resource comparison**

- The app is a simple "Hello World" API built with .NET 7.
- The measurements are taken using [Microsoft Crank](https://github.com/dotnet/crank)

Crank is the benchmarking infrastructure used by the .NET team to run benchmarks including scenarios from the TechEmpower Web Framework Benchmarks.



# **Security issues**

Chiselled Ubuntu images are designed with security in mind. Their reduced size greatly diminishes the attack surface of the images, making them less likely to be affected by vulnerabilities.

Also keep in mind that they do not include a package manager or shell, which completely disarms certain classes of attacks, but that also means that you won't be able to login into a running container and start executing shell commands like you can do with regular images.   

Running scripts or commands in a running container is not considered a good security practice. However, if you are currently doing it, please be aware that by default, you won't be able to do so with this type of image.

## **Vulnerability scanning comparison**

Just like in the previous section let's compare the vulnerabilities between a "Hello World" app built with .NET 8 using Debian 12 (Bookworm) as the base image and another one using Ubuntu Chiseled (Jammy) as the base image. 

Nowadays there are numerous container security scanners available (in one of my previous [posts](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/) I discussed some of the well-known options and how to integrate them with Azure Pipelines). For this comparison, we will be using:
- **Clair via AWS ECR** 
- **Trivy**

# **Migrate a containerized app to Chiseled Ubuntu containers**

For this section, I'll try to migrate a WebApi that is using a Debian base image in its Dockerfile, more specifically  the ``runtime-deps:7.0-bullseye-slim`` image, to use a Chiseled Ubuntu image as its base image.


## **Application**

The app can be found here:
- https://github.com/karlospn/opentelemetry-metrics-demo

It is a bookStore API built using .NET 7. It allows us to do the following actions:
- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and delete orders.

## **Migration process**




# **Conclusion**