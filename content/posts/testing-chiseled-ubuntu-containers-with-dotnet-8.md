---
title: "Using Chiseled Ubuntu containers images with .NET"
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

<add-img>


# **.NET in Chiseled Ubuntu Containers**

Right now (06/06/2023), Chiseled Ubuntu container images are available for .NET 6, .NET 7 and .NET 8, but are still in preview.

The .NET 6 and .NET 7 images can be found only in the nightly repositories while still in preview.

<add-links>

The .NET 8 images are also still in preview (.NET 8 has not been released yet), but can be found in the official repositories.

<add-links>

The next image shows a comparison between an .NET app using a Ubuntu image vs a Chiseled Ubuntu image

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

The use of a non-root user is not unique to .NET Chiseled Ubuntu containers. Starting from .NET 8, all images, regardless of the distro, will transition to using a non-root user.


# **Comparison between Chiseled Ubuntu containers and Debian 12 containers**

The default operating system for working with containers in .NET has always been Debian. When we create a new Dockerfile for our app from Visual Studio, it always generates a Dockerfile that uses Debian as the base image, regardless of whether the app is built with .NET Core 3.1, .NET 5, .NET 7, or .NET 8.

Therefore, I believe that making a comparison between a "Hello World" app built with .NET 8 using Debian 12 (Bookworm) as the base image and another one using Ubuntu Chiseled (Jammy) as the base image could be interesting.

# **Security issues**

Chiselled Ubuntu images are designed with security in mind. Their reduced size greatly diminishes the attack surface of the images, making them less likely to be affected by vulnerabilities.

Also keep in mind that they do not include a package manager or shell, which completely disarms certain classes of attacks, but that also means that you won't be able to login into a running container and start executing shell commands like you can do with regular images.   

Running scripts or commands in a running container is not considered a good security practice. However, if you are currently doing it, please be aware that by default, you won't be able to do so with this type of image.

## **Vulnerability scanning comparison**

Just like in the previous section let's compare the vulnerabilities between a "Hello World" app built with .NET 8 using Debian 12 (Bookworm) as the base image and another one using Ubuntu Chiseled (Jammy) as the base image. 

Nowadays there are numerous container security scanners available, in one of my previous [posts](https://www.mytechramblings.com/posts/testing-container-vulnerabilities-scanners-using-azure-pipelines/) I discussed some of the well-known options and how to integrate them with Azure Pipelines. For this comparison, we will use:
- **Clair via AWS ECS** 
- **Trivy**

# Migrate a containerized app to start using Chiseled Ubuntu containers
