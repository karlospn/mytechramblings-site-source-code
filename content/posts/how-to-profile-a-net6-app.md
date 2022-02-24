---
title: "Profiling a .NET6 app running in a docker container with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio"
date: 2022-02-22T18:49:37+01:00
description: "This post will demonstrate how you can do profiling with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio on a .NET6 application."
tags: ["csharp", "dotnet", "profiling"]
draft: true
---

# Introduction

Every software developer at some time or another has stumbled with an application that doesn't perform well. And when it happens I find that quite a few people are unfamiliar about what steps should they take to solve them.    
And that's what I decided to write this beginner's article. 

I have built a .NET6 application beforehand, this app has a few performance issues and in this post we're going to try to spot them.

The performance issues on the demo application are oversimplified and I'm sure that if you take a quick glance on its source code you'll be able to spot them pretty quickly, but we are not going to do that, instead of that we're going to use the .NET CLI diagnostic tools to profile the app and try to understand what is happening.  

On a real scenario the performance issues will not be so easy to spot on, but the objective of this post is to serve as stepping stone for beginners, so when they face a real performance problem they know what are the basic steps that should follow.

Most of the topics I’m going to tackle in this demo are basic stuff:

- How to monitor the application performance counters and use it as a first-level performance investigation.   
- How to create a memory dump to investigate why are the threads being blocked.
- How to investigate the contents of the memory heap to find out why the size keeps growing.
- How to obtain a cpu trace to investigate why the CPU percentage counter is spiking.

So, if you’re a performance veteran this post is not for you.

# .NET CLI diagnostic tools

A couple years ago Microsoft introduced a series of new diagnostic tools:

- ``dotnet-counters`` to view Performance Counters.
- ``dotnet-dump`` to capture and analyze Windows and Linux dumps.
- ``dotnet-trace`` to capture runtime events, GC collections and sample CPU stacks.
- ``dotnet-gcdump`` to collect GC dumps.

Those tools are cross-platform and nowadays are the preferred method of collecting diagnostic information for .NET Core scenarios targeting .NET Core 3.0 or above.

# Adding .NET CLI tools in a container

The .NET Core global CLI diagnostic tools (dotnet-counters, dotnet-dump, dotnet-gcdump, and dotnet-trace) are designed to work in a wide variety of environments and should all work directly in Docker containers.

The only complicating factor of using these tools in a container is that they are installed with the .NET SDK and many Docker containers run without the .NET SDK present.   

One easy solution to this problem is to install the tools in the initial Docker image. The tools don't need the .NET SDK to run, only to be installed. Therefore, it's possible to create a Dockerfile with a multi-stage build that installs the tools in a build stage (where the .NET SDK is present) and then copies the binaries into the final image.   

The only downside to this approach is increased Docker image size.


```yaml
FROM mcr.microsoft.com/dotnet/sdk:6.0-bullseye-slim AS build-env
WORKDIR /app

# Copy csproj and restore dependencies
COPY Profiling.Api.csproj ./src/
RUN dotnet restore "./src/Profiling.Api.csproj"

# Copy everything, build and publish
COPY . ./src/
RUN dotnet publish src/*.csproj -c Release -o /app/publish

# Install dotnet debug tools
RUN dotnet tool install --tool-path /tools dotnet-trace \
 && dotnet tool install --tool-path /tools dotnet-counters \
 && dotnet tool install --tool-path /tools dotnet-dump

# Build runtime imagedock
FROM mcr.microsoft.com/dotnet/aspnet:6.0-bullseye-slim

# Copy dotnet-tools
WORKDIR /tools
COPY --from=build-env /tools .

# Copy app
WORKDIR /app
COPY --from=build-env /app/publish .

# Set entrypoint
ENTRYPOINT ["dotnet", "Profiling.Api.dll"]
```

# Executing .NET CLI tools in a running container

In order to access these tools at runtime, we need to be able to access the container at runtime.   
We can use the ``docker exec`` command to launch a shell in the running container.

``docker exec -it -w //tools <container-id> sh``

And once we're inside the container we're ready to execute any of the diagnostic tools like this:

``./dotnet-counters monitor --process-id 1``


# Demo Application

The application we're going to profile is a .NET 6 API with 3 endpoints:

- ``/blocking-threads`` endpoint.
- ``/high-cpu`` endpoint.
- ``/memory-leak`` endpoint.

As you can see, the performance issues we will tackle on each endpoint are pretty self-explanatory.

I'll be running the app as a docker container on my local machine because it's the easiest and fastest setup possible.   

It doesn't matter if you're running your app on a k8s cluster, a virtual machine or any other cloud services, the steps you need to follow when trying to pinpoint a performance issue will be exactly the same, the only thing that its going to change is how to distribute the diagnostic tools binaries. 

# Profiling the ``/blocking-threads`` endpoint

# Profiling the ``/high-cpu`` endpoint

# Profiling the ``/memory-leak`` endpoint

