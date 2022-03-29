---
title: "Profiling a .NET6 app running on AWS ECS Fargate with dotnet-monitor"
date: 2022-03-28T22:01:20+02:00
tags: ["aws", "fargate", "ecs", "containers", "dotnet", "csharp", "performance"]
description: "In this post I want to show you how to deploy and profile a .NET6 application that is running on AWS ECS Fargate with dotnet-monitor as a sidecar container."
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/profiling-net6-api-on-aws-ecs-fargate-using-dotnet-monitor).

# **Notice**

This post might look the same as my [last one]((https://www.mytechramblings.com/posts/profiling-a-net-app-with-dotnet-cli-diagnostic-tools/)), even the performance problems I'm going to try to pinpoint in the demo app are going to be the same ones
  
The main difference lies in the fact that in my last post I was using the .NET CLI diagnostic tools to profile an app and in this one I'll be using the ``dotnet-monitor`` tool.

The demo app will going to profile is going to be hosted on AWS ECS Fargate, so I'll show you the necessary steps you need to follow to deploy a .NET6 application on Fargate with ``dotnet-monitor`` as a sidecar container.  

# **Introduction**

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

# **dotnet-monitor**

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


# **Demo Application**

The application I'm going to profile is the same one I used on my [last post](https://www.mytechramblings.com/posts/profiling-a-net-app-with-dotnet-cli-diagnostic-tools/).

I decided to use the same application so you can see that the steps needed to profile an app with ``dotnet-monitor`` and .NET CLI diagnostic tools are basically the same.

The demo app is a .NET 6 API with 3 endpoints:

- ``/blocking-threads`` endpoint.
- ``/high-cpu`` endpoint.
- ``/memory-leak`` endpoint.
  
But first of all, we need to deploy the app into AWS ECS Fargate with ``dotnet-monitor`` as a sidecar container.

# **Deploy into AWS ECS Fargate**

``dotnet-monitor`` can be deployed as a sidecar container to collect diagnostic information from other containers running .NET.

This is the entire stack what we're going to deploy on AWS:

![fargate-dotnet-monitor-cdk-app](/img/fargate-dotnet-monitor-cdk-app.png.png)

- A VPC with 2 public Subnets.
- A public Application Load Balancer that listens on port 80.
  - This ALB is used to access the NET6 Web API.
  - You can use port 443 but I didn't have an SSL certificate at hand.
- A public Application Load Balancer that listens on port 52323
  - This one is used to interact with ``dotnet-monitor`` 
  - You could use a single ALB and set 2 listeners with 2 target groups, but in a realistic scenario you  don't want to open a port on your internet facing ALB, instead you'll create a secondary internal ALB and use it to interact with the ``dotnet-monitor`` API.
- An ECS Task Definition with 2 containers: API container + dotnet-monitor container.

The next code snippet is probably the most interesting part of the CDK app, and shows how to setup properly the ``dotmnet-monitor`` container and the app container in the ECS Task Definition.

```csharp
  private FargateTaskDefinition CreateTaskDefinition()
        {
            var task = new FargateTaskDefinition(this,
                "task-definition-ecs-profiling-dotnet-demo",
                new FargateTaskDefinitionProps
                {
                    Cpu = 1024,
                    Family = "task-definition-ecs-profiling-dotnet-demo",
                    MemoryLimitMiB = 2048
                });

            task.AddVolume(new Amazon.CDK.AWS.ECS.Volume
            {
                Name = "diagnostics"
            });

            task.AddVolume(new Amazon.CDK.AWS.ECS.Volume
            {
                Name = "dumps"
            });

            var linuxParams = new LinuxParameters(this, "sys-ptrace-linux-params");
            linuxParams.AddCapabilities(Capability.SYS_PTRACE);

            task.AddContainer("container-app",
                new ContainerDefinitionOptions
                {
                    Cpu = 512,
                    MemoryLimitMiB = 1024,
                    Image = ContainerImage.FromAsset("../src/Profiling.Api"),
                    LinuxParameters = linuxParams,
                    Environment = new Dictionary<string, string>
                    {
                        {"DOTNET_DiagnosticPorts","/diag/port,nosuspend,connect"}
                    },
                    Logging = LogDriver.AwsLogs(new AwsLogDriverProps
                    {
                        StreamPrefix = "ecs"
                    }),
                    PortMappings = new IPortMapping[]
                    {
                        new PortMapping
                        {
                            ContainerPort = 80
                        }
                    }
                }).AddMountPoints(new MountPoint
            {
                ContainerPath = "/diag",
                SourceVolume = "diagnostics"
            }, new MountPoint
            {
                ContainerPath = "/dumps",
                SourceVolume = "dumps",
                ReadOnly = false
            });

            task.AddContainer("dotnet-monitor",
                new ContainerDefinitionOptions
                {
                    Cpu = 256,
                    MemoryLimitMiB = 512,
                    Image = ContainerImage.FromRegistry("mcr.microsoft.com/dotnet/monitor:6"),
                    Environment = new Dictionary<string, string>
                    {
                        { "DOTNETMONITOR_DiagnosticPort__ConnectionMode", "Listen" },
                        { "DOTNETMONITOR_DiagnosticPort__EndpointName", "/diag/port" },
                        { "DOTNETMONITOR_Urls", "http://+:52323" },
                        { "DOTNETMONITOR_Storage__DumpTempFolder", "/dumps"}
                    },
                    Command = new[] { "--no-auth" },
                    PortMappings = new IPortMapping[]
                    {
                        new PortMapping
                        {
                            ContainerPort = 52323
                        }
                    }

                }).AddMountPoints(new MountPoint
            {
                ContainerPath = "/diag",
                SourceVolume = "diagnostics"
            }, new MountPoint
            {
                ContainerPath = "/dumps",
                SourceVolume = "dumps",
                ReadOnly = false
            });

            return task;
        }
```
- ``dotnet-monitor`` tool is configured to run in Listen mode. The tool establishes a diagnostic communication channel at the specified Unix Domain Socket path by the ``DOTNETMONITOR_DiagnosticPort__EndpointName`` environment variable.   
Also the application container has a ``DOTNET_DiagnosticPorts`` environment variable specified so that the application's runtime will communicate with the dotnet monitor instance at the specified Unix Domain Socket path.    
Each port configuration specifies whether it is a ``suspend`` or ``nosuspend`` port. Ports specifying suspend in their configuration will cause the runtime to pause early on in the startup path before most runtime subsystems have started. This allows any agent to receive a connection and properly setup before the application startup continues
  
- In Listen mode the Unix Domain Socket file needs to be shared among the ``dotnet-monitor`` container and the app container you wish to monitor. If your diagnostic port is established at ``/diag/port``, then a shared volume needs to be mounted at ``/diag`` in the ``dotnet-monitor`` container and mounted at ``/diag`` in the app container. In Fargate to share a directory we'll use [Bind Mounts](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/bind-mounts.html).

- ``dotnet-monitor`` tool is configured to instruct the application runtime to produce dump files at the path specified by the ``DOTNETMONITOR_Storage__DumpTempFolder`` environment variable. If you want to create a dump on Fargate you'll need to enable the ``SYS_PTRACE`` Capability on the application container.
  
- By default, ``dotnet monitor``  requires authentication. For the purposes of this demo, authentication has been disabled by using the ``--no-auth`` command line switch. 
  
- By default, ``dotnet monitor``  has HTTPS enabled. For the purposes of this example, the artifact URLs have been changed to ``http://localhost:52323`` using the ``DOTNETMONITOR_Urls`` environment variable such that a TLS certificate is not required.

If you want to take a look at the entire CDK app, you can go to my [Github](https://github.com/karlospn/profiling-net6-api-on-aws-ecs-fargate-using-dotnet-monitor) page.

After deploying the CDK app into AWS, we can test the ``dotnet-monitor`` API with a simple cURL command:

```bash
$ curl http://alb-mon-ecs-prf-dotnet-demo-744303317.eu-west-1.elb.amazonaws.com:52323/info
{"version":"6.1.0+48853dad0d5d1efd50a90904505f68a137c7b789","runtimeVersion":"6.0.3","diagnosticPortMode":"Listen","diagnosticPortName":"/diag/port"}
```

# **Profiling the API using dotnet-monitor**

In the following sections I'm going to do exactly the same performance investigations I did in my [last post](https://www.mytechramblings.com/posts/profiling-a-net-app-with-dotnet-cli-diagnostic-tools/).

The main difference is going to be that in my last post I was using the .NET CLI diagnostic tools and in this one I'll be using ``dotnet-monitor.``

So depending on your needs you can go read this post or the other one.

# **Profiling the /blocking-threads endpoint**

## **1. Using the trace endpoint from dotnet-monitor to investigate possible performance issues**

Observing the performance counters is always the first step when we want to begin a performance investigation.

To obtain the performance counters in a given interval we're going invoke the ``dotnet-monitor`` ``/trace`` endpoint.

This endpoint captures a diagnostic trace of a process based on a predefined set of trace profiles.
The format is the following one:

```bash
GET /trace?pid={pid}&profile={profile}&durationSeconds={durationSeconds}&egressProvider={egressProvider}
```

- ``pid``: The ID of the process.
- ``profile``: The name of the profile(s) used to collect events. Multiple profiles may be specified by separating them with commas. Default is ``Cpu,Http,Metrics``**
- ``durationSeconds``: The duration of the trace operation in seconds. Default is 30
- ``egressProvider``: If specified, uses the named egress provider for egressing the collected trace. When not specified, the trace is written to the HTTP response stream.

**The available profiles are:
- ``Cpu``: Tracks CPU usage and general .NET runtime information.
- ``Http``: Tracks ASP.NET request handling and HttpClient requests.
- ``Logs``: Tracks log events emitted at the Debug log level or higher.
- ``Metrics``: Tracks event counters from the System.Runtime, Microsoft.AspNetCore.Hosting, and Grpc.AspNetCore.Server event sources.

If we invoke the ``/trace`` endpoint, we will receive a ``.nettrace`` file back. If we open it with Visual Studio, it will look like this:

![dotnetmonitor-nettrace-output.png](/img/dotnetmonitor-nettrace-output.png)

As you can see the contents of the ``.nettrace`` file are exactly the same as the output of the ``dotnet-counters`` tool. The main difference is that with ``dotnet-counters`` we can watch the application performance counters in real-time, meanwhile with ``dotnet-monitor`` we need to sample the counters during a period of time before being able to analyze them.

There quite a few performce counters in the ``.nettrace`` file, let me explain briefly the meaning of each one of them.

- **Current Requests**:  The total number of requests that have started, but not yet stopped.
- **Failed Requests**:  The total number of failed requests that have occurred for the life of the app.
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
- **Gen 2 GC Count**: The number of times GC has occurred for Gen 2 per update interval.
- **Gen 2 Size**:  The number of bytes for Gen 2 GC.
- **IL Bytes Jitted**: The total size of ILs that are JIT-compiled, in bytes.
- **LOH Size**: The number of bytes for the Large Object Heap.
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


Right now there is no traffic in our application, so nothing worth mentioning is happening. Let's start by applying some load.   
To generate some load on the app I'll be using [Bombardier](https://github.com/codesenberg/bombardier).   
Bombardier is a modern HTTP(S) benchmarking tool, written in Golang and really simple to use when you want to run a performance or a load test.

I'm going to apply a load of 100 requests per second during 120 seconds to the ``blocking-threads`` endpoint.   
``bombardier -c 120 --rate 100 -l -d 120s http://alb-pub-ecs-prf-dotnet-demo-100430270.eu-west-1.elb.amazonaws.com/blocking-threads``

And at the same time I'm going to capture a trace using the ``dotnet-monitor /trace`` endpoint.  
``GET http://alb-mon-ecs-prf-dotnet-demo-744303317.eu-west-1.elb.amazonaws.com:52323/trace?pid=1&durationSeconds=120
``
The following two screenshots show the values of the counters during the load test:

[dotnetmonitor-counters-1](/img/dotnetmonitor-counters-1.png)

[dotnetmonitor-counters-2](/img/dotnetmonitor-counters-2.png)

The counters that seemed OK were:
- The CPU usage was low.
- Very few GC collections and no Gen 2 GC Collection.
- LOH had a low and steady size.

The counters that seemed OFF were:
- Poor Request/Rate ratio.
- The **"ThreadPool Thread Count"** counter was increasing exponentially during the test duration.

The "ThreadPool Thread Count" should not be growing like this, in an ideal application  everything runs asynchronously which means that a low number of threads can serve a huge amount of requests.

In an async world when an incoming request arrives it gets a new thread from the Threadpool, that thread runs it’s current operation and when an async method is awaited, saves the current context and adds itself to the pool as an available thread.   
When the async call completes or if a new request comes in at this point, the Threadpool checks if all the threads are already in operation, if yes it spawns up a new thread, if not it takes up one of the available threads from the pool.

The fact that the "ThreadPool Thread Count" counter keeps growing means that the application is expanding the Threadpool to handle incoming requests, which means that probably some threads are being blocked somewhere.    

The blockage could be on purpose and maybe the app is doing some long running task, but maybe the threads are being blocked because there is something wrong, so it's worth investigating further.

## **2. Using the dump endpoint from dotnet-monitor and Visual Studio to analyze possible blocked threads**

Now we can open it with Visual Studio. To start a debugging session, select "Debug with Managed Only".

![vs-dump-debug](/img/vs-dump-debug.png)

Keep in mind that a dump it's a snapshot of your application in a concrete point in time, so you can't debug the application, but what we can do is select "Debug > Windows > Threads" to get a  list of what were the threads doing at the time we captured the dump.


And as you can see on the next screenshot, it looks suspicious the amount of threads that are on the "Sleep" state.

![vs-dump-threads-view](/img/vs-dump-threads-view.png)

Another thing we can do is select "Debug > Windows > Parallel Stacks" and it gives us a more graphical view of what's happening with the active threads.

![vs-dump-parallel-stacks](/img/vs-dump-parallel-stacks.png)

In this case we can see that there are 74 threads grouped on 2 stacks, if we take a deeper look at those stacks we recognize that "BlockingThreadsService.Run" is a function of our app, if we double click on it, Visual Studio will take us there.

> *To navigate from the Parallel Stacks window to the source code we need to enable the microsoft symbols server and also have access to the application symbols.*

After double-clicking on the "BlockingThreadsService.Run" on the Parallel Stacks windows, Visual Studio will show us which command was this thread executing when the dump was created.

![vs-dump-task-waitall](/img/vs-dump-task-waitall.png)

And it seems that the thread was stopped in the "Task.WaitAll" command.

If we keep diggingg a little bit further and take a look at this thread call stack, we will see that Task1 is blocked on the lock statement

![vs-dump-task-lock](/img/vs-dump-task-lock.png)

It's pretty clear that there is some bad code on the "BlockingThreadsService.Run" function that is blocking threads, so now we can try to fix it.    
I'm not going to fix it in this post, we have found the performance issue and now we can move on to another one.
