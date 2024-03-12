---
title: "Building and deploying a .NET 8 App on ARM64 processors with Azure Pipelines and AWS ECS Fargate. Part 2: Demo"
date: 2024-03-01T09:42:30+01:00
tags: ["dotnet", "containers", "azure", "aws", "devops", "arm64"]
description: "TBD"
draft: true
---

If you examine the enhancements in the last .NET versions, you'll see that each one of them brings quite a few improvements for ARM64 processors. The argument for using ARM64 processors over AMD64 is that ARM64 processors are cheaper, more efficient, and can reduce the carbon footprint.

But, how easy is it to work with them in .NET? Let's do a little test in this post. Let's check out the process of building and deploying a simple .NET 8 API to an ARM64 processor.

I don't have an ARM64 processor at hand right now, so I decided to use the following products:
- **AWS ECR** to store the container image.
- **AWS ECS Fargate** to run the API. 
- **Azure Pipelines** to build and publish the image into the Amazon Registry.

Why use AWS instead of Azure? Nothing in particular, in my last posts i was using Azure, so it might be a nice change of scenery to use AWS.


# **The Application**

It is a simple .NET 8 API with an endpoint that counts how many prime numbers are in a given range.

I decided to use a more CPU-bound method instead of using a "Hello World" API, so down the line, we can compare performance between the API running on Fargate in an ARM64 processor and the same API running on Fargate on an AMD64 processor.

The next code snippet shows exactly what the API does. It implements a version of the Sieve of Eratosthenes algorithm to find all prime numbers up to a given number.

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
            var n = 100000000;
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

This is the initial Dockerfile we're going to use. It is a simple one, it uses the ASP.NET base image with an Ubuntu distro, and it runs the ``dotnet restore``, ``dotnet build``, and ``dotnet publish`` commands.

Right now, it is using an AMD64 base image, in the next section, we're going to change that.

```yml
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy AS build

WORKDIR /app
COPY . ./

RUN dotnet restore "./Arm64Testing.WebApi.csproj" \
    --runtime linux-x64

RUN dotnet build "./Arm64Testing.WebApi.csproj" \
    -c Release \
    --runtime linux-x64 \
    --no-restore

RUN dotnet publish "./Arm64Testing.WebApi.csproj" \
    -c Release \
    -o /app/publish \
    --runtime linux-x64 \
    --no-restore \
    --no-build

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy
EXPOSE 8080
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

# **Preparing the API to work with an ARM64 processor and pushing it into ECR**

Preparing the API to work with an ARM64 processor and pushing it into ECR involves a series of steps. 

Here's a basic outline:
- **Update Dockerfile**: Update your Dockerfile to build the application for an ARM64 architecture.
- **Build the Docker Image**: Use Docker to build the Docker image for your .NET API. Make sure to tag the image appropriately.
- **Login to ECR**: Use the AWS CLI to authenticate Docker to your ECR repository.
- **Tag the Image**: Tag your Docker image with the ECR repository URI.
- **Push the Image to ECR**: Push the Docker image to your ECR repository.

## **Updating the Dockerfile**

The first step we need to take is to modify the Dockerfile so it can build and run the API on an ARM64 processor.

Microsoft provides specific images for ARM64 processors for both the .NET SDK and the .NET runtime. These images have the same name but the tags end with the ``-arm64v8`` suffix.

We also need to update the ``--runtime`` parameter we're using in the ``dotnet restore``, ``dotnet build``, and ``dotnet publish`` commands.   
This parameter is used to specify the target architecture, identifying the platform where the application will run.


In some Dockeriles, instead of the ``--runtime`` parameter, you might find the ``--arch`` parameter or the ``--os`` parameter. All three parameters serve the same purpose.

The following code snippet shows the resulting Dockerfile after applying the changes to make it compatible with an ARM64 processor. Essentially, the changes include:

- Changing the base image from the build step from ``mcr.microsoft.com/dotnet/sdk:8.0-jammy`` to ``mcr.microsoft.com/dotnet/sdk:8.0-jammy-arm64v8``

- Changing the base image from the run step from ``mcr.microsoft.com/dotnet/aspnet:8.0-jammy`` to ``mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8``

- Changing the ``--runtime`` parameter value of the ``dotnet restore``, ``dotnet build`` and ``dotnet publish`` commands from ``linux-x64`` to ``linux-arm64``.

```yml
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy-arm64v8 AS build

WORKDIR /app
COPY . ./

RUN dotnet restore "./Arm64Testing.WebApi.csproj" \
    --runtime linux-arm64

RUN dotnet build "./Arm64Testing.WebApi.csproj" \
    -c Release \
    --runtime linux-arm64 \
    --no-restore

RUN dotnet publish "./Arm64Testing.WebApi.csproj" \
    -c Release \
    -o /app/publish \
    --runtime linux-arm64 \
    --no-restore \
    --no-build

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-arm64v8
EXPOSE 8080
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Arm64Testing.WebApi.dll"]
```

### **Updating the Dockerfile for multi-architecture**

## **Building the container image using Azure Pipelines**
