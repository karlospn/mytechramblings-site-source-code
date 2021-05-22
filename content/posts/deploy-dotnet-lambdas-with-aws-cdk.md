---
title: "Different ways to deploy a .NET Lambda using AWS CDK"
date: 2021-05-22T16:09:46+02:00
draft: true
---

I already talked in some of my post about AWS CDK, but for those unaware of AWS CDK is one of many framework that allow us to define cloud infrastructure in code.

AWS CDK is available in a bunch of different languages:

- JavaScript
- TypeScript 
- Python 
- Java 
- .NET Core

I already shared my thoughts about AWS CDK in some of my older posts so in this one I will cut to the chase. In this post I will focus on showing different options available on AWS CDK when you want to deploy a .NET Core Lambda.


To begin with I think that there are 2 concepts worth mentioning:

## Deployment Types

Right now if you want to deploy a Lambda you have two types of deployments available: container images and .zip files.

A container image includes the base operating system, the runtime, Lambda extensions, your application code and its dependencies.   
You upload your container images to Amazon Elastic Container Registry (Amazon ECR). To deploy the image to your function, you specify the Amazon ECR image URL using the Lambda console, the Lambda API, command line tools, or the AWS SDKs.

A .zip file archive includes your application code and its dependencies. When you author functions using the Lambda console or a toolkit, Lambda automatically creates a .zip file archive of your code.   
You can upload a .zip file as your deployment package using the Lambda console, AWS Command Line Interface (AWS CLI), or to an Amazon Simple Storage Service (Amazon S3) bucket.


## Dotnet Managed runtime

Lambda supports multiple languages through the use of runtimes.   
The runtime provides a language-specific environment that runs in an execution environment. The runtime relays invocation events, context information, and responses between Lambda and the function. You can use runtimes that Lambda provides, or build your own.

The policy with .NET runtimes with AWS has always been to support only LTS versions for managed runtimes. So you cannot use .NET 5 as a managed runtime.   

What exactly it means? It means that if you want to deploy a NET5 lambda you have 2 options available:
- Use the container image deployment type.
- Or if you really want to deploy your lambda as a .zip file you're going to need to build a custom NET5 runtime.

If you don't mind working with .NET Core 3.1 instead of .NET5 you can use the .zip file deployment straightaway.

---
<br/>

Without further ado let me show you 3 different options available with AWS CDK when you want to deploy a .NET Core Lambda.

- **How to deploy a NET5 Lambda using a container image and with AWS CDK Bundling options.**
- **How to deploy a NET Core 3.1 Lambda using a .zip file and with AWS CDK Bundling options.**
- **How to deploy a NET Core 3.1 Lambda using a pre-built .zip file.**

> In the following sections I'm going to assume you have a dotnet lambda already built and you want to deploy it.


# 1. Deploy a NET5 Lambda using a container image and with AWS CDK Bundling options

The first step is going to be to create the Dockerfile.   

AWS has its own dotnet lambda image so you just need a NET 5 SDK image to build and publish the project and push the artifact into the NET5 lambda image.

```yaml
FROM public.ecr.aws/lambda/dotnet:5.0 as base

FROM mcr.microsoft.com/dotnet/sdk:5.0 as build-image
WORKDIR /app

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish --no-restore -c Release -o /publish

FROM base AS final
WORKDIR /var/task
COPY --from=build-image /publish .
CMD ["My.Net5.Lambda::My.Net5.Lambda.Function::FunctionHandler"]
```



# 2. Deploy a NET Core 3.1 Lambda using a .zip file and with AWS CDK Bundling options



# 3. Deploy a NET Core 3.1 Lambda using a pre-built .zip file

