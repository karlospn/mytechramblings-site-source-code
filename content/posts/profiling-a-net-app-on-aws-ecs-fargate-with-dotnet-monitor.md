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

AWS ECS Fargate is one of this environments. Fargate is a serverless container platform, it is a general-purpose platform, which means it doesn't have a set of diagnostic tools for .NET put in place and trying to access inside a running container is practically a nightmare, that's where the ``dotnet-monitor``  tool becames really handy.

In this post I'll show you how to deploy a .NET6 application on AWS ECS Fargate with ``dotnet-monitor`` as a sidecar container, and afterwards how to use the ``dotnet-monitor`` to do a profile.

# dotnet-monitor

``dotnet monitor`` is a tool that provides an unified way to collect diagnostics.

It exposes an HTTP API for on demand collection of artifacts. You can call the API endpoints to collect traces, dumps, gcdumps and metrics.
