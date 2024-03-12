---
title: "Building and deploying a .NET 8 App on an ARM64 processor using Azure Pipelines and AWS ECS Fargate. Part 2: Demo"
date: 2024-03-01T09:42:30+01:00
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

This application requires a SQL Server database, but to simplify, we will use EF with an in-memory database.

- You can find the source code [here](https://github.com/karlospn/opentelemetry-metrics-demo)

# **Building the Dockerfile**

## **Using emulation**

## **Using cross-compilation**

# **Building the container image using Azure Pipelines (and pushing it into ECR)**

## **Using emulation**

## **Using cross-compilation**

# **Creating the AWS infrastructure**

# **Testing the app**

# **Benchmark comparing the app running on an ARM64 ECS Fargate service versus the same app running on an AMD64 ECS Fargate service**
