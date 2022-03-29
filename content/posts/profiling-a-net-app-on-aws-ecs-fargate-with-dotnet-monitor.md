---
title: "Profiling a .NET6 app running on AWS ECS Fargate with dotnet-monitor"
date: 2022-03-28T22:01:20+02:00
tags: ["aws", "fargate", "ecs", "containers", "dotnet", "csharp", "performance"]
description: "In this post I want to show you how to deploy and profile a .NET6 application that is running on AWS ECS Fargate with dotnet-monitor as a sidecar container."
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/profiling-net6-api-on-aws-ecs-fargate-using-dotnet-monitor).

I'm pretty sure that everyone at some time or other has stumbled with an application that performs well on your local machine, maybe even on the dev environments, but when you put it on production it begins to perform poorly, and it becomes an ordeal trying to pinpoint the problem.

In my [last post](https://www.mytechramblings.com/posts/profiling-a-net-app-with-dotnet-cli-diagnostic-tools/) I talked about how to profile a .NET6 application using the .NET CLI diagnostic tools.

Those tools (dotnet-counters, dotnet-dump, dotnet-gcdump and dotnet-trace) are designed to work in a wide variety of environments.   
For exmaple, if your apps are running on a VM you can add them on the base image or connect remotely to the machine via SSH and install them.    
If you're running on some sort of Kubernetes cluster you can bake them inside the docker image and afterwards connect to the pod using the kubectl command line.   
There are even some Cloud Services, like Azure App Services, that uses its own set of diagnostic tools, so you don't need them at all.

Nonetheless there are some environments where collecting diagnostics can become a pain in the ass, for profiling an app in these kind of environments you can use the ``dotnet-monitor`` tool.

AWS ECS Fargate is one of this environments. Fargate is a serverless container platform, it is a general-purpose platform, which means it doesn't have a specific set of diagnostic tools for .NET put in place and also trying to access a running container is a nightmare, that's why the ``dotnet-monitor``  is a perfect fit to obtain diagnostics from an app running on Fargate.


**This post is more oriented towards the necessary steps to deploy a .NET6 application on AWS ECS Fargate with ``dotnet-monitor`` as a sidecar container rather than how to profile an app.**   
**But nonetheless after setting everything up on the AWS side I plan to show you how to use the ``dotnet-monitor`` HTTP API to profile an app.**

# dotnet-monitor

``dotnet monitor`` provides an unified way to collect traces, memory dumps, gc dumps and metrics.

It exposes an HTTP API for on demand collection of artifacts with the following endpoints:

- ``/processes``: Gets detailed information about discoverable processes.
- ``/dump``: Captures managed dumps of processes without using a debugger.
- ``/gcdump``: Captures GC dumps of processes.
- ``/trace``: Captures traces of processes without using a profiler.
- ``/metrics``: Captures a snapshot of metrics of the default process in the Prometheus exposition format.
- ``/livemetrics``: Captures live streaming metrics of a process.
- ``/logs``: Captures logs of processes.
- ``/info`` Gets info about dotnet monitor.
- ``/operations``: Gets egress operation status or cancels operations.

To put it simply, ``dotnet-monitor`` just exposes the .NET CLI diagnostics tools via an HTTP API.

Right now, it is available via two different distribution mechanisms:

- As a .NET Core global tool.
- As a container image available via the Microsoft Container Registry (MCR). 
 
``dotnet-monitor`` is not only for running on-demand profiling, it can also be configured to automatically collect diagnostic artifacts when certain conditions are met within a process. For example, if collect a CPU dump when you have a sustained high CPU.


# Demo Application

The application I'm going to profile is the same one I used on my [last post](https://www.mytechramblings.com/posts/profiling-a-net-app-with-dotnet-cli-diagnostic-tools/).

I decided to use the same application so you can see that the steps needed to profile an app with ``dotnet-monitor`` and .NET CLI diagnostic tools are basically the same.

The demo app is a .NET 6 API with 3 endpoints:

- ``/blocking-threads`` endpoint.
- ``/high-cpu`` endpoint.
- ``/memory-leak`` endpoint.
  
But first of all, we need to deploy the app into AWS ECS Fargate with ``dotnet-monitor`` as a sidecar container.

# Deploy the app and dotnet-monitor into AWS ECS Fargate

``dotnet-monitor`` can be deployed as a sidecar container to collect diagnostic information from other containers running .NET Core 3.1 or later processes. That's the reason why is a perfect fit to use it on Fargate.

# Profiling the /blocking-threads endpoint

# Profiling the /high-cpu endpoint

# Profiling the /memory-leak endpoint
