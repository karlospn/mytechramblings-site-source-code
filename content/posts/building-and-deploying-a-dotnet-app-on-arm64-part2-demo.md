---
title: "Building and deploying a .NET 8 App on an ARM64 processor using Azure Pipelines and AWS ECS Fargate. Part 2: Demo"
date: 2024-04-02T09:42:30+01:00
tags: ["dotnet", "containers", "azure", "aws", "devops", "arm64"]
description: "In this two-part series, I’m going to show you how to build and deploy a .NET 8 app container image that targets an ARM64 processor. In part 2, we will attempt to perform an end-to-end process. We will build a .NET 8 API, containerize the app targeting an ARM64 processor using Azure Pipelines, and finally deploy it on AWS ECS Fargate.    
Additionally, we will conduct a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container."
draft: true
---

> This is a two-part series post.
> - **Part 1**: Key concepts about how to build multi-platform images.
> - **Part 2**: A practical example of how to build a container image targeting an ARM64 processor using Azure Pipelines and how to deploy it on AWS ECS Fargate.    
It will also include a quick benchmark to compare the performance of the application running on an ARM64 Fargate container against the same app using an AMD64 Fargate container.

In the first part of this blog post, we discussed how to create a multi-platform image, how multi-platform .NET images work, and which options are available when we want to build a multi-platform image (emulation, cross-compilation, or using a host with the target architecture).

Now, it's time to build an end-to-end process. Let's attempt to build a container image targeting ARM64 using a CI runner, deploy the app into an ARM64 machine host, and test that it's working correctly.

I don't have an ARM64 machine at hand right now, so to build the E2E process, I decided to use the following cloud services:

- **AWS ECR** to store the container image.
- **AWS ECS Fargate** to run the API.
- **Azure Pipelines** to build and publish the image into the Amazon Registry.
- **Artillery** to test the API.

Why use AWS instead of Azure? Nothing in particular. In my last posts, I was using Azure, so it might be a nice change of scenery to use AWS.

# **Application**

The application we’re going to use is a BookStore API built using .NET 8.

The API can perform the following actions:

- Get, add, update and delete book categories.
- Get, add, update and delete books.
- Get, add, update and delete inventory.
- Get, add and update orders.

This application requires a SQL Server database, but to simplify it, we will use **EF with an in-memory database**.

```csharp
services.AddDbContext<BookStoreDbContext>(options =>
{
    options.UseInMemoryDatabase("BookStoreDb");
});
```

- You can find the source code [here](https://github.com/karlospn/opentelemetry-metrics-demo)

# **Building the Dockerfile**

Instead of building a single multi-platform Dockerfile, let's make the effort to build two: one that uses emulation and another one with cross-compilation.

## **Using emulation**

To create a multi-platform Dockerfile that uses emulation, we don't need to do any extra step; just a simple run-of-the-mill Dockerfile will suffice.

Emulation is the easiest option of the three available, because it requires no changes at all to your Dockerfile. The BuildKit automatically detects the secondary architectures that are available and when BuildKit needs to run a binary for a different architecture, it automatically loads it.

The following code snippet shows the Dockerfile with the following features:
- It is a multi-stage Dockerfile: it uses an Ubuntu 22.04 as a base image for the build stage and an Ubuntu Chiseled image for the runtime stage.
- The app is published as self-contained, resulting in an application bundle that includes the .NET runtime, libraries, as well as the application itself and its dependencies.

```yml
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy AS build-env
WORKDIR /app

# Copy everything
COPY . ./

# Restore packages
RUN dotnet restore -s "https://api.nuget.org/v3/index.json"

# Build project
RUN dotnet build "./src/BookStore.WebApi/BookStore.WebApi.csproj" \ 
    -c Release \
	--self-contained true

# Publish app
RUN dotnet publish "./src/BookStore.WebApi/BookStore.WebApi.csproj" \
	-c Release \
	-o /app/publish \
	--no-restore \ 
	--no-build \
	--self-contained true

# Build runtime image
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0.0-jammy-chiseled-extra

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Starts on port 8080
ENV ASPNETCORE_URLS=http://+:8080

# Set Entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```

Now, let's try to build the container image on my AMD64 machine. Remember that we're running the ``dotnet build`` command on an AMD64 host machine and we're targeting an ARM64 processor, which means that it will use the [QEMU](https://www.qemu.org/) emulator for building the resulting container image.


Now, let's attempt to build the container image on my AMD64 machine. Remember that we're executing the ``docker build`` command on an AMD64 host machine while targeting an ARM64 processor. This implies that it will utilize the QEMU emulator to construct the resulting container image.

```script
docker build --platform=linux/arm64 -t bookstore-api:arm64-emulation -f src/BookStore.WebApi/Dockerfile .
[+] Building 132.1s (15/15) FINISHED
```
It takes quite some time (and consumes a significant amount of CPU and memory resources), but we eventually end up with a functional ARM64 container image.

## **Using Cross-Compilation**

The idea behind Cross-Compilation is to utilize a multi-stage build Dockerfile. In the build stage, you compile your code for the target architecture, and in the run stage, you configure the runtime to be exported to the final image.

Using the Dockerfile from the previous section as a starting point, there are several modifications needed to make it compatible with Cross-Compilation.

1. Use the pre-defined build argument ``BUILDPLATFORM`` to pin the builder to use the host's native architecture as the build platform. This is necesarry to prevent emulation.  
In simpler terms, we need to add ``--platform=$BUILDPLATFORM`` into the ``FROM`` instruction of the build stage.

2. Add the ``ARG TARGETARCH`` instructions for the build stage, making the ``TARGETARCH`` build arguments available to the commands in this stage.

3. Modify the ``dotnet restore``, ``dotnet build`` and ``dotnet publish`` commands so they generate the application binaries for the target architecture (in our case ``arm64``).    
To accomplish this, we're going to use the ``--arch`` attribute along with the ``TARGETARCH`` argument.

```yml
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0-jammy AS build-env
WORKDIR /app
ARG TARGETARCH

# Copy everything
COPY . ./

# Restore packages
RUN dotnet restore -s "https://api.nuget.org/v3/index.json" \
	-a $TARGETARCH

# Build project
RUN dotnet build "./src/BookStore.WebApi/BookStore.WebApi.csproj" \ 
    -c Release \
	-a $TARGETARCH \ 
	--self-contained true

# Publish app
RUN dotnet publish "./src/BookStore.WebApi/BookStore.WebApi.csproj" \
	-c Release \
	-o /app/publish \
	--no-restore \ 
	--no-build \
	--self-contained true \
	-a $TARGETARCH

# Build runtime image
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0.0-jammy-chiseled-extra

# Copy artifact
WORKDIR /app
COPY --from=build-env /app/publish .

# Starts on port 8080
ENV ASPNETCORE_URLS=http://+:8080

# Set Entrypoint
ENTRYPOINT ["./BookStore.WebApi"]
```
Now, let's attempt to build the container image targeting an ARM64 processor.

```script
$ docker build --platform=linux/arm64 -t bookstore-api:arm64-cc -f src/BookStore.WebApi/Dockerfile .
[+] Building 16.2s (15/15) FINISHED
```
Using cross-compilation, we've reduced the build time 130 seconds to 17 seconds, and the CPU usage from nearly 90% to no more than 20%.

# **Building the container image using Azure Pipelines (and pushing it into ECR)**

In the previous section, we have created a couple of Dockerfiles (one that uses emulation to build the container image and another one that uses Cross Compilation) and we have test both of them on my local machine.    
That's not a very realistic scenario. Now, instead of using my computer, let's attempt to build them using an Azure Pipelines hosted agent.

## **Using emulation**

Real straightforward. We're using an Ubuntu hosted agent, the Dockerfile from the previous section and on the pipeline we're running only a ``docker build`` command using the ``--platform`` attribute.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker build --platform=linux/arm64 -t bookstore-api:arm64-emu -f src/BookStore.WebApi/Dockerfile .
```

And it throws an error.

```text
#11 [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
#11 0.229 exec /bin/sh: exec format error
#11 ERROR: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 1
------
 > [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json":
0.229 exec /bin/sh: exec format error
------
Dockerfile:8
--------------------
   6 |     
   7 |     # Restore packages
   8 | >>> RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
   9 |     
  10 |     # Build project
--------------------
ERROR: failed to solve: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 1
```

Thi is because the Azure Pipelines linux hosting agent doesn't have QEMU installed, QEMU only comes preinstalled with Docker desktop, if you're using Docker Engine you'll need to install it.

To install it, we're going to use the [qemu-user-static](https://github.com/multiarch/qemu-user-static) image. This image installs every necessary thing to run QEMU.

Let's try it again, but installing QEMU first.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
      docker build --platform=linux/arm64 -t bookstore-api:arm64-emu -f src/BookStore.WebApi/Dockerfile .
```

And it throws another error. This time is a segmentation fault error.

```text

#11 [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
#11 22.03   Determining projects to restore...
#11 26.85 Segmentation fault (core dumped)
#11 ERROR: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 139
------
 > [build-env 4/6] RUN dotnet restore -s "https://api.nuget.org/v3/index.json":
22.03   Determining projects to restore...
26.85 Segmentation fault (core dumped)
------
Dockerfile:8
--------------------
   6 |     
   7 |     # Restore packages
   8 | >>> RUN dotnet restore -s "https://api.nuget.org/v3/index.json"
   9 |     
  10 |     # Build project
--------------------
ERROR: failed to solve: process "/bin/sh -c dotnet restore -s \"https://api.nuget.org/v3/index.json\"" did not complete successfully: exit code: 139
```

As I told you in part 1 of this blog post, the Docker Engine + QEMU + .NET doesn't really work well together. 

I'm going to abandon the idea of tryin to build a image using Emulation, is not worth it having to deal with the emulator inconsistencies when working with .NET. And anyway using Cross-Compilation is the way to go here, we were just messing with emulation to see if it was a viable option or not. And the answer to that question is that it is not a viable option.

## **Using cross-compilation**

Same thing as the last section. We're using an Ubuntu hosted agent, the Dockerfile from the previous section and on the pipeline we're running only a ``docker build`` command using the ``--platform`` attribute.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker build --platform=linux/arm64 -t bookstore-api:arm64-cc -f src/BookStore.WebApi/Dockerfile .
```

And it works without any issue. Now let's push it into AWS ECR and in the next section we'll test it using AWS ECS Fargate.

To push it into AWS ECR, let's keep it simple. I'm going to use the Azure DevOps [AWSShellScript](https://docs.aws.amazon.com/vsts/latest/userguide/awsshell.html) to do it. This task runs a shell script using bash with AWS credentials. The script will do the following steps:
- Retrieve an authentication token and authenticate the Docker client with my ECR registry.
- Create a new ECR repository.
- Tag the container image so I can push it into the repository.
- Push the image to my newly created ECR repository.

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

steps:
- task: Bash@3
  inputs:
    targetType: 'inline'
    script: |  
      docker build --platform=linux/arm64 -t bookstore-api:arm64-cc -f src/BookStore.WebApi/Dockerfile .

- task: AWSShellScript@1
  inputs:
    awsCredentials: 'aws-dev'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 156267164149.dkr.ecr.eu-west-1.amazonaws.com
      aws ecr create-repository --repository-name arm64-bookstore-api
      docker tag bookstore-api:arm64-cc 156267164149.dkr.ecr.eu-west-1.amazonaws.com/arm64-bookstore-api:cc
      docker push 156267164149.dkr.ecr.eu-west-1.amazonaws.com/arm64-bookstore-api:cc
```

# **Creating the AWS infrastructure**

I'm going to need the following resources:

- A VPC.
- An ALB.
- An ECS Cluster.
- An ECS Task Definition + An ECS Service.

And to create them, I'll be using [AWS CDK](https://aws.amazon.com/cdk/).

The only thing worth showing in here is how to create an ECS Task Definition that targets an ARM64 processor, and it is as simple as setting the ``CpuArchitecture`` attribute to ``ARM64``, and that's it. 

The rest of the resources don't care if you're deploying an app that targets an ARM64 processor or and AMD one.

```csharp
private FargateTaskDefinition CreateTaskDefinition()
{
  var repository = Repository.FromRepositoryName(this, "arm64-bookstore-repository", "arm64-bookstore-api");

  var task = new FargateTaskDefinition(this,
      "task-definition-bookstore-api",
      new FargateTaskDefinitionProps
      {
          Cpu = 512,
          Family = "task-definition-bookstore-api",
          MemoryLimitMiB = 1024,
          RuntimePlatform = new RuntimePlatform
          {
              CpuArchitecture = CpuArchitecture.ARM64,
              OperatingSystemFamily = OperatingSystemFamily.LINUX
          }
      });

  task.AddContainer("bookstore-api",
      new ContainerDefinitionOptions
      {
          Cpu = 512,
          MemoryLimitMiB = 1024,
          Image = ContainerImage.FromEcrRepository(repository, "cc"),
          Logging = LogDriver.AwsLogs(new AwsLogDriverProps
          {
              StreamPrefix = "ecs"
          }),
          PortMappings = new IPortMapping[]
          {
              new PortMapping
              {
                  ContainerPort = 8080
              }
          }
      });

  return task;
}
```

# **Benchmark**

After deploying the Bookstore API on AWS Fargate, I want to do a little benchmark so I can compare how the app fares when running on ARM64 versus when it run on an AMD64 processor. To achieve it, I have deployed a second bookstore API but this one targets an AMD64 processor.

Now, let's run a little benchmark. I'm going to use [Artillery](https://www.artillery.io/) as the load generator.

## Test 1: 
- Duration: 180 seconds.
- Rate: 50 req/sec.
- Endpoint: GET ``/api/books``

### **Requests**

- ARM64 BookStore API
```text
http.codes.200: ................................................................ 9000
http.downloaded_bytes: ......................................................... 45000000
http.request_rate: ............................................................. 50/sec
http.requests: ................................................................. 9000
http.response_time:
  min: ......................................................................... 43
  max: ......................................................................... 1447
  mean: ........................................................................ 61.3
  median: ...................................................................... 56.3
  p95: ......................................................................... 80.6
  p99: ......................................................................... 113.3
http.responses: ................................................................ 9000
vusers.completed: .............................................................. 9000
vusers.created: ................................................................ 9000
vusers.created_by_name.0: ...................................................... 9000
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 86.3
  max: ......................................................................... 1771.6
  mean: ........................................................................ 125.2
  median: ...................................................................... 113.3
  p95: ......................................................................... 149.9
  p99: ......................................................................... 284.3
```
- AMD64 BookStore API
```text
http.codes.200: ................................................................ 9000
http.downloaded_bytes: ......................................................... 45000000
http.request_rate: ............................................................. 50/sec
http.requests: ................................................................. 9000
http.response_time:
  min: ......................................................................... 44
  max: ......................................................................... 5347
  mean: ........................................................................ 62.8
  median: ...................................................................... 56.3
  p95: ......................................................................... 82.3
  p99: ......................................................................... 111.1
http.responses: ................................................................ 9000
vusers.completed: .............................................................. 9000
vusers.created: ................................................................ 9000
vusers.created_by_name.0: ...................................................... 9000
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 88.1
  max: ......................................................................... 7209.7
  mean: ........................................................................ 238.6
  median: ...................................................................... 113.3
  p95: ......................................................................... 162.4
  p99: ......................................................................... 5378.9
```

### **CPU usage**

![arm64-test1-cpu-comparison](/img/arm64-test1-cpu-comparison.png.png)


### **Memory usage**

![arm64-test1-mem-comparison](/img/arm64-test1-mem-comparison.png)


## Test 2: 
- Duration: 60 seconds.
- Rate: 100 req/sec.
- Endpoint: GET ``/api/categories``

### **Requests**

- ARM64 BookStore API
```text
http.codes.200: ................................................................ 6000
http.downloaded_bytes: ......................................................... 990000
http.request_rate: ............................................................. 100/sec
http.requests: ................................................................. 6000
http.response_time:
  min: ......................................................................... 43
  max: ......................................................................... 540
  mean: ........................................................................ 63.2
  median: ...................................................................... 58.6
  p95: ......................................................................... 96.6
  p99: ......................................................................... 117.9
http.responses: ................................................................ 6000
vusers.completed: .............................................................. 6000
vusers.created: ................................................................ 6000
vusers.created_by_name.0: ...................................................... 6000
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 87.8
  max: ......................................................................... 1343.7
  mean: ........................................................................ 127.6
  median: ...................................................................... 117.9
  p95: ......................................................................... 172.5
  p99: ......................................................................... 219.2
```
- AMD64 BookStore API
```text
http.codes.200: ................................................................ 6000
http.downloaded_bytes: ......................................................... 990000
http.request_rate: ............................................................. 100/sec
http.requests: ................................................................. 6000
http.response_time:
  min: ......................................................................... 44
  max: ......................................................................... 168
  mean: ........................................................................ 62
  median: ...................................................................... 58.6
  p95: ......................................................................... 87.4
  p99: ......................................................................... 113.3
http.responses: ................................................................ 6000
vusers.completed: .............................................................. 6000
vusers.created: ................................................................ 6000
vusers.created_by_name.0: ...................................................... 6000
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 87.7
  max: ......................................................................... 279.4
  mean: ........................................................................ 122.4
  median: ...................................................................... 115.6
  p95: ......................................................................... 165.7
  p99: ......................................................................... 194.4
```

### **CPU usage**

![arm64-test2-cpu-comparison](/img/arm64-test2-cpu-comparison.png.png)

### **Memory usage**

![arm64-test2-mem-comparison](/img/arm64-test2-mem-comparison.png)

## Test 3: 
- Duration: 120 seconds.
- Rate: 80 req/sec.
- Endpoint: GET ``/api/orders``

### **Requests**

- ARM64 BookStore API
```text
http.codes.200: ................................................................ 9600
http.downloaded_bytes: ......................................................... 18124800
http.request_rate: ............................................................. 80/sec
http.requests: ................................................................. 9600
http.response_time:
  min: ......................................................................... 44
  max: ......................................................................... 159
  mean: ........................................................................ 64
  median: ...................................................................... 61
  p95: ......................................................................... 87.4
  p99: ......................................................................... 108.9
http.responses: ................................................................ 9600
vusers.completed: .............................................................. 9600
vusers.created: ................................................................ 9600
vusers.created_by_name.0: ...................................................... 9600
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 87.6
  max: ......................................................................... 233.1
  mean: ........................................................................ 123.5
  median: ...................................................................... 122.7
  p95: ......................................................................... 153
  p99: ......................................................................... 186.8
```
- AMD64 BookStore API
```text
http.codes.200: ................................................................ 9600
http.downloaded_bytes: ......................................................... 18124800
http.request_rate: ............................................................. 80/sec
http.requests: ................................................................. 9600
http.response_time:
  min: ......................................................................... 44
  max: ......................................................................... 304
  mean: ........................................................................ 63.9
  median: ...................................................................... 61
  p95: ......................................................................... 87.4
  p99: ......................................................................... 104.6
http.responses: ................................................................ 9600
vusers.completed: .............................................................. 9600
vusers.created: ................................................................ 9600
vusers.created_by_name.0: ...................................................... 9600
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 88.3
  max: ......................................................................... 409.9
  mean: ........................................................................ 124.8
  median: ...................................................................... 122.7
  p95: ......................................................................... 153
  p99: ......................................................................... 183.1
```
### **CPU usage**

![arm64-test3-cpu-comparison](/img/arm64-test3-cpu-comparison.png.png)

### **Memory usage**

![arm64-test3-mem-comparison](/img/arm64-test3-mem-comparison.png)


To be fair the bookstore API is probably not very well suited for comparing how a .NET 8 app runs on Fargate when targeting an ARM64 processor versus an AMD64 processor.

I have written another .NET 8 Api. It implements a version of the Sieve of Eratosthenes algorithm to find all prime numbers up to a given number.
The next code snippet shows exactly what the API does.

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Collections;

namespace Arm64Testing.WebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class PrimeController : ControllerBase
    {

        [HttpGet]
        public int Get()
        {
            var n = 10000000;
            return CountPrimes(n);
        }

        public static int CountPrimes(int n)
        {
            int segmentSize = 10000;
            bool[] isPrime = new bool[segmentSize];

            int count = 0;
            for (int low = 2; low <= n; low += segmentSize)
            {
                int high = Math.Min(low + segmentSize - 1, n);
                for (int i = 0; i < segmentSize; i++)
                {
                    isPrime[i] = true;
                }

                int sqrtHigh = (int)Math.Sqrt(high);
                for (int p = 2; p <= sqrtHigh; p++)
                {
                    if (IsPrime(p))
                    {
                        int start = Math.Max(p * p,
                            (low + p - 1) / p * p);
                        for (int j = start; j <= high; j += p)
                        {
                            isPrime[j - low] = false;
                        }
                    }
                }

                for (int i = Math.Max(2, low); i <= high; i++)
                {
                    if (isPrime[i - low])
                    {
                        count++;
                    }
                }
            }

            return count;
        }

        public static bool IsPrime(int num)
        {
            if (num < 2)
                return false;

            for (int i = 2; i * i <= num; i++)
            {
                if (num % i == 0)
                    return false;
            }

            return true;
        }
    }
}
```

This is a more CPU-bound API, so I'm going to deploy this app in 2 Fargate services: the first one will target an ARM64 processor and the second one an AMD64 processor.

Let's run a test with Artillery during 240 seconds with 3req/sec (more than that it will crash the app, because it takes quite considerable time to calculate the result).

### **Requests**

- ARM64 BookStore API
```text
http.codes.200: ................................................................ 720
http.downloaded_bytes: ......................................................... 4320
http.request_rate: ............................................................. 3/sec
http.requests: ................................................................. 720
http.response_time:
  min: ......................................................................... 182
  max: ......................................................................... 622
  mean: ........................................................................ 405.5
  median: ...................................................................... 441.5
  p95: ......................................................................... 528.6
  p99: ......................................................................... 561.2
http.responses: ................................................................ 720
vusers.completed: .............................................................. 720
vusers.created: ................................................................ 720
vusers.created_by_name.0: ...................................................... 720
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 229.6
  max: ......................................................................... 682.8
  mean: ........................................................................ 467.7
  median: ...................................................................... 507.8
  p95: ......................................................................... 584.2
  p99: ......................................................................... 632.8
```
- AMD64 BookStore API
```text
http.codes.200: ................................................................ 720
http.downloaded_bytes: ......................................................... 4320
http.request_rate: ............................................................. 2/sec
http.requests: ................................................................. 720
http.response_time:
  min: ......................................................................... 193
  max: ......................................................................... 1355
  mean: ........................................................................ 573.5
  median: ...................................................................... 596
  p95: ......................................................................... 820.7
  p99: ......................................................................... 944
http.responses: ................................................................ 720
vusers.completed: .............................................................. 720
vusers.created: ................................................................ 720
vusers.created_by_name.0: ...................................................... 720
vusers.failed: ................................................................. 0
vusers.session_length:
  min: ......................................................................... 245.4
  max: ......................................................................... 2172.2
  mean: ........................................................................ 641.4
  median: ...................................................................... 658.6
  p95: ......................................................................... 907
  p99: ......................................................................... 1107.9
```
### **CPU usage**

![arm64-prime-test-cpu-comparison](/img/arm64-prime-test-cpu-comparison.png.png)

