---
title: "Profiling a .NET6 app running in a docker container with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio"
date: 2022-02-22T18:49:37+01:00
description: "This post will demonstrate how you can do profiling with dotnet-trace, dotnet-dump, dotnet-counters and Visual Studio on a .NET6 application."
tags: ["csharp", "dotnet", "profiling"]
draft: true
---

# Introduction

Every software developer at some time or another has stumbled with an application that doesn't perform well. And when it happens I find a lot of unfamiliarity about what steps should take to solve them.    
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
The source code can be found on my [GitHub](https://github.com/karlospn/profiling-net6-api-demo)

I'll be running the app as a docker container on my local machine because it's the easiest and fastest setup possible.    
It doesn't matter if you're running your app on a k8s cluster, a virtual machine or any other cloud services, the steps you need to follow when trying to pinpoint a performance issue will be exactly the same, the only thing that its going to change is how to distribute the diagnostic tools binaries. 

To simulate a more realistic environment I have set some memory an Cpu limits to the app running on docker.   
To be more precise, a **1024Mb memory limit and a 1 CPUs limit**.

``docker run -d -p 5003:80 -m 1024m --cpus=1 profiling.api``

Also I'll be using [bombardier](https://github.com/codesenberg/bombardier) to generate traffic on the app.   
Bombardier is a modern HTTP(S) benchmarking tool, written in Go language and really easy to use for performance and load testing.

# Profiling the ``/blocking-threads`` endpoint

First thing we're going to do is executing the ``dotnet-counters`` command tool.

``dotnet-counters`` is **always the first step** when we want to begin a performance investigation.   

``dotnet-counters`` is a performance monitoring and first-level performance investigation tool. It can observe performance counter values that are published via the EventCounter API.    
For example, you can quickly monitor things like the CPU usage or the rate of exceptions being thrown in your .NET Core application to see if there's anything suspicious before diving into more serious performance investigation using ``dotnet-dump`` or ``dotnet-trace``.

The command we're going to use to launch ``dotnet-counters`` is the following one:

``./dotnet-counters monitor --process-id 1 --counters System.Runtime,Microsoft.AspNetCore.Hosting``

- The ``monitor`` command starts monitoring a .NET application.
- The ``-p|--process-id`` parameter is a required one and specifies the ID of the process we want to monitor, in this case we're inside a docker container so the PID is always 1.   
If you're running outside a container and do not know the PID of the procss you want to monitor you can run the ``./dotnet-counters ps`` command.
- The ``--counters`` parameter is an optional one and you need to specify a comma-separated list of counters.   
The default counters are the ``System.Runtime`` counters, but when working with an API I find useful to monitor also the ``Microsoft.AspNetCore.Hosting`` counters, so you can see how many requests are being served, how many requests are failing, request rate, etc.

That's the output you'll see after launching the command:

```bash
[Microsoft.AspNetCore.Hosting]
    Current Requests                                               0
    Failed Requests                                                0
    Request Rate (Count / 1 sec)                                   0
    Total Requests                                                 0
[System.Runtime]
    % Time in GC since last GC (%)                                 0
    Allocation Rate (B / 1 sec)                                8,168
    CPU Usage (%)                                                  0
    Exception Count (Count / 1 sec)                                0
    GC Committed Bytes (MB)                                        0
    GC Fragmentation (%)                                           0
    GC Heap Size (MB)                                              3
    Gen 0 GC Count (Count / 1 sec)                                 0
    Gen 0 Size (B)                                                 0
    Gen 1 GC Count (Count / 1 sec)                                 0
    Gen 1 Size (B)                                                 0
    Gen 2 GC Count (Count / 1 sec)                                 0
    Gen 2 Size (B)                                                 0
    IL Bytes Jitted (B)                                       68,801
    LOH Size (B)                                                   0
    Monitor Lock Contention Count (Count / 1 sec)                  0
    Number of Active Timers                                        0
    Number of Assemblies Loaded                                  110
    Number of Methods Jitted                                     620
    POH (Pinned Object Heap) Size (B)                              0
    ThreadPool Completed Work Item Count (Count / 1 sec)           0
    ThreadPool Queue Length                                        0
    ThreadPool Thread Count                                        0
    Time spent in JIT (ms / 1 sec)                                 0
    Working Set (MB)                                              61
```
As you can see, it's quite a lot of counters, let me explain briefly the meaning of each one of them.

- **Current Requests**:  The total number of requests that have started, but not yet stopped.
- **Faied Requests**:  The total number of failed requests that have occurred for the life of the app.
- **Request Rate**: The number of requests that occur per update interval.
- **Total Requests**:  The total number of requests that have occurred for the life of the app.
- **% Time in GC since last GC (%)**: The percent of time in GC since the last GC.
- **Allocation Rate**:  Number of bytes allocated in the managed heap between update intervals.
- **CPU Usage**: The percentage of CPU usage relative to to CPU resource allocated.
- **Exception Count**: The number of exceptions that have occurred.
- **GC Commited Bytes**: The number of bytes committed by the GC.
- **GC Fragmentation**: The GC Heap Fragmentation (available on .NET 5 and later versions).
- **GC Heap Size**: The number of bytes allocated by the GC.
- **Gen 0 GC Count**: The number of times GC has occurred for Gen 0 per update interval.
- **Gen 0 Size**:  The number of bytes for Gen 0 GC.
- **Gen 1 GC Count**: The number of times GC has occurred for Gen 1 per update interval.
- **Gen 1 Size**:  The number of bytes for Gen 1 GC.
- **Gen 2 GC Count**: Number of Gen 2 GCs between update intervals
- **Gen 2 Size**:  The number of times GC has occurred for Gen 2 per update interval.
- **IL Bytes Jitted**: The total size of ILs that are JIT-compiled, in bytes.
- **LOH Size**: The number of bytes for the large object heap.
- **Monitor Lock Contention Count**: The number of times there was contention when trying to take the monitor's lock.
- **Number of Active Timers**: The number of Timer instances that are currently active.
- **Number of Assemblies Loaded**: The number of Assembly instances loaded into a process at a point in time.
- **Number of Methods Jitted**:  The number of methods that are JIT-compiled.
- **POH (Pinned Object Heap) Size**: The number of bytes for the pinned object heap (available on .NET 5 and later versions).
- **ThreadPool Completed Work Item Count**: The number of work items that have been processed so far in the ThreadPool.
- **ThreadPool Queue Length**: The number of work items that are currently queued to be processed in the ThreadPool.
- **ThreadPool Thread Count**: The number of thread pool threads that currently exist in the ThreadPool.
- **Time spent in JIT**: The total time spent doing jitting work.
- **Working Set**: The amount of physical memory mapped to the process.

Right now we're monitoring the app counters in real time, but there is no traffic in our application and nothing worth mentioning is happening, so let's apply some load using bombardier.

I'm going to apply a load of 100 requests per second during 120 seconds to the ``blocking-threads`` endpoint.

``./bombardier.exe -c 50 --rate 50 -l -d 120s https://localhost:5003/blocking-threads``

Here's how the counters look like after 60 seconds
```bash
[Microsoft.AspNetCore.Hosting]
    Current Requests                                              15
    Failed Requests                                                0
    Request Rate (Count / 1 sec)                                  14
    Total Requests                                               325
[System.Runtime]
    % Time in GC since last GC (%)                                 0
    Allocation Rate (B / 1 sec)                              449,624
    CPU Usage (%)                                                  0
    Exception Count (Count / 1 sec)                                0
    GC Committed Bytes (MB)                                       16
    GC Fragmentation (%)                                          22.432
    GC Heap Size (MB)                                             12
    Gen 0 GC Count (Count / 1 sec)                                 1
    Gen 0 Size (B)                                         3,640,560
    Gen 1 GC Count (Count / 1 sec)                                 0
    Gen 1 Size (B)                                         3,488,928
    Gen 2 GC Count (Count / 1 sec)                                 0
    Gen 2 Size (B)                                         4,720,320
    IL Bytes Jitted (B)                                      269,978
    LOH Size (B)                                              98,408
    Monitor Lock Contention Count (Count / 1 sec)                  0
    Number of Active Timers                                        0
    Number of Assemblies Loaded                                  113
    Number of Methods Jitted                                   2,938
    POH (Pinned Object Heap) Size (B)                      4,443,728
    ThreadPool Completed Work Item Count (Count / 1 sec)          43
    ThreadPool Queue Length                                    1,619
    ThreadPool Thread Count                                       32
    Time spent in JIT (ms / 1 sec)                                 0
    Working Set (MB)                                              83
```
And here's how the counters look like after 120 second (end of the load test)
```bash
[Microsoft.AspNetCore.Hosting]
    Current Requests                                              73
    Failed Requests                                                0
    Request Rate (Count / 1 sec)                                  56
    Total Requests                                             2,727
[System.Runtime]
    % Time in GC since last GC (%)                                 0
    Allocation Rate (B / 1 sec)                            2,476,392
    CPU Usage (%)                                                  1
    Exception Count (Count / 1 sec)                                0
    GC Committed Bytes (MB)                                       27
    GC Fragmentation (%)                                          12.394
    GC Heap Size (MB)                                             23
    Gen 0 GC Count (Count / 1 sec)                                 0
    Gen 0 Size (B)                                                24
    Gen 1 GC Count (Count / 1 sec)                                 0
    Gen 1 Size (B)                                         3,823,928
    Gen 2 GC Count (Count / 1 sec)                                 0
    Gen 2 Size (B)                                        15,022,080
    IL Bytes Jitted (B)                                      272,879
    LOH Size (B)                                              98,408
    Monitor Lock Contention Count (Count / 1 sec)                  2
    Number of Active Timers                                        0
    Number of Assemblies Loaded                                  113
    Number of Methods Jitted                                   2,974
    POH (Pinned Object Heap) Size (B)                      4,443,728
    ThreadPool Completed Work Item Count (Count / 1 sec)         201
    ThreadPool Queue Length                                    1,990
    ThreadPool Thread Count                                       81
    Time spent in JIT (ms / 1 sec)                                 0
    Working Set (MB)                                              90
```

During this test the CPU usage was low, allocation rate also was low, there were almost no GC happening and the LOH keeps a steady size.    

The only thing interesting is that the **"ThreadPool Queue Length"** and **"ThreadPool Thread Count"** counters were growing exponentially during the test duration.

The "ThreadPool Thread Count" should not be growing like this, in an ideal application  everything runs asynchronously which means that a low number of threads can serve a huge amount of requests.

In an async world when an incoming request arrives it spawns a new thread, that thread runs it’s current operation and when an async method is awaited, saves the current context and adds itself to the global pool as an available thread.   
When the  async call completes or if a new request comes in at this point, the Threadpool checks if all the threads are already in operation, if yes it spawns up a new thread, if not it takes up one of the available threads from the pool.

The fact that the Threadpool keeps growing means that the application is expanding the Threadpool to handle incoming requests which means that probably some threads are being blocked somewhere.    

This blockage could be on purpose and maybe the app is doing some long running task, but maybe the threads are being blocked because there is something not right on the application, so it's worth investigating further.


# Profiling the ``/high-cpu`` endpoint

# Profiling the ``/memory-leak`` endpoint

