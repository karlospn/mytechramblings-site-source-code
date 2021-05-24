---
title: "4 ways to deploy a .NET Core Lambda using AWS CDK"
date: 2021-05-25T10:01:46+02:00
tags: ["aws", "lambda", "cdk", "dotnet"]
draft: true
---

> **Show me the code**   
> If you don't care about the post I have upload the code on my [Github](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk)

I already talked in some of my post about AWS CDK, but for those unaware of AWS CDK is a framework that allows us to define cloud infrastructure in code.

AWS CDK is available in a bunch of different languages:

- JavaScript
- TypeScript 
- Python 
- Java 
- .NET Core

I already shared my thoughts about AWS CDK in some of my older posts so in this one I will cut to the chase and I will focus on showing you a few options available when you want to deploy a .NET Core Lambda.

Before start showing you the code I think that there are 2 concepts worth mentioning:

## Deployment Types

Right now if you want to deploy a Lambda you have **two types of deployments available**: **container images** and **.zip files**.

A container image includes the base operating system, the runtime, Lambda extensions, your application code and its dependencies.   
You upload your container images to Amazon Elastic Container Registry (Amazon ECR).    
To deploy the image to your function, you specify the Amazon ECR image URL using the Lambda console, the Lambda API, command line tools, or the AWS SDKs.

A .zip file archive includes your application code and its dependencies. When you author functions using the Lambda console or a toolkit, Lambda automatically creates a .zip file archive of your code.   
You can upload a .zip file as your deployment package using the Lambda console, AWS Command Line Interface (AWS CLI), or to an Amazon Simple Storage Service (Amazon S3) bucket.


## Dotnet Managed runtime

Lambda supports multiple languages through the use of runtimes.   
The runtime provides a language-specific environment that runs in an execution environment. The runtime relays invocation events, context information, and responses between Lambda and the function. You can use managed runtimes that Lambda provides or you can build your own.

The policy with .NET runtimes has always been to support only LTS versions for managed runtimes. So you cannot use .NET 5 as a managed runtime.   

So if you want to deploy a .NET 5 lambda you have 2 options available:
- Use the container image deployment type.
- Or if you really want to deploy your lambda as a .zip file you're going to need to build a custom .NET 5 runtime.

Also if you don't mind working with .NET Core 3.1 instead of .NET 5 you can use the .zip file deployment straightaway.

---
<br/>

The 4 options to deploy a .NETCore lambda with AWS CDK that I'm going to show you in this post are the following ones:

- **Deploy a NET 5 lambda using a container image and the AWS CDK ``FromImageAsset`` option.**
- **Deploy a NET 5 lambda using a container image that has already been pushed into an ECR registry.**
- **Deploy a NET Core 3.1 lambda using a .zip file and the AWS CDK ``FromAsset`` with bundling options.**
- **Deploy a NET Core 3.1 lambda using an existing .zip file and the AWS CDK ``FromAsset`` option.**


> In the following sections I'm going to assume that you have already a NET 5 lambda and a NET Core 3.1 lambda already built so I can focus on how to deploy them.

# 1. Deploy a NET 5 lambda using a container image and the AWS CDK ``FromImageAsset`` option

> _You can find a working example of this scenario in this [Link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net5BundlingContainerLambdaCdk)_

In this scenario the AWS CDK and the application will be deployed at the same time.    

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;

namespace Net5BundlingContainerLambdaCdk
{
    public class Net5LambdaCdkStack : Stack
    {
        internal Net5LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {
            DockerImageCode dockerImageCode = DockerImageCode.FromImageAsset("../My.Net5.Lambda");

            DockerImageFunction dockerImageFunction = new DockerImageFunction(this, 
                "container-image-lambda-function",
                new DockerImageFunctionProps()
            {
                Code = dockerImageCode,
                Description = ".NET 5 Docker Lambda function",
                
            });
        }
    }
}
```

The ``DockerImageCode.FromImageAsset`` construct needs to point to a local folder with a 
Dockerfile.   
The Dockerfile output will be used as the function handler.

In this particular example when we execute the ``cdk deploy`` command the following steps are going to happen:
- The container is going to be built using the dockerfile found in the ``../My.Net5.Lambda``
- An ECR repository named ``aws-cdk\assets`` will be created if it doesn't exists yet (the **``aws-cdk\assets`` is a centralized repository**, so every lambda created using this method will be pushed into this very same ECR repository).
- The image is going to be pushed into the ``aws-cdk/assets`` repository.
- The lambda will be created among the rest of the infrastructura declared in the CDK project.


Also let me show you how the dockerfile looks like:

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

AWS offers a .NET 5 lambda base image so I'm using the .NET 5 SDK image to build and publish the project and afterwards I'm pushing the artifact into the NET 5 lambda base image.

# 2. Deploy a NET 5 lambda using a container image that has already been pushed into an ECR registry

> _You can find a working example of this scenario in this [Link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net5ExistingContainerLambdaCdk)_

In this scenario the AWS CDK and the application won't be deployed at the same time. This scenario assumes that you have already pushed the lambda image into an axisting ECR repository.

```csharp

using Amazon.CDK;
using Amazon.CDK.AWS.ECR;
using Amazon.CDK.AWS.Lambda;

namespace Net5ExistingContainerLambdaCdk
{
    public class Net5LambdaCdkStack : Stack
    {
        internal Net5LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {

            var ecrRepo = Repository.FromRepositoryName(this, 
                "ecr-image-repository", 
                "net5.container.lambda");

            DockerImageCode dockerImageCode = DockerImageCode.FromEcr(ecrRepo);
            
            DockerImageFunction dockerImageFunction = new DockerImageFunction(this, 
                "container-image-lambda-function", 
                new DockerImageFunctionProps()
            {
                Code = dockerImageCode,
                Description = ".NET 5 Docker Lambda function"
            });
        }
    }
}

```
 
The ``DockerImageCode.FromEcr`` command allows us to specify an image that already exists in ECR as the lambda function handler.

In this particular example when we execute the ``cdk deploy`` command the following steps are going to happen:
- The lambda will be created using the image stored in the ``net5.container.lambda`` ECR repository.



# 3. Deploy a NET Core 3.1 lambda using a .zip file and the AWS CDK ``FromAsset`` with bundling options

> _You can find a working example of this scenario in this [Link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/NetCore31BundlingZipFileLambdaCdk)_


In this scenario the AWS CDK and the application will be deployed at the same time.    

```csharp
  
using System.Collections.Generic;
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using AssetOptions = Amazon.CDK.AWS.S3.Assets.AssetOptions;

namespace NetCore31BundlingZipFileLambdaCdk
{
    public class NetCore31LambdaCdkStack : Stack
    {
        internal NetCore31LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {
            IEnumerable<string?> commands = new[]
            {
                "cd /asset-input",
                "export DOTNET_CLI_HOME=\"/tmp/DOTNET_CLI_HOME\"",
                "export PATH=\"$PATH:/tmp/DOTNET_CLI_HOME/.dotnet/tools\"",
                "dotnet tool install -g Amazon.Lambda.Tools",
                "dotnet lambda package -o output.zip",
                "unzip -o -d /asset-output output.zip"
            };

            Function function = new Function(this,
                "zip-lambda-function",
                new FunctionProps
            {
                Runtime = Runtime.DOTNET_CORE_3_1,
                Code = Code.FromAsset("../My.NetCore31.Lambda", new AssetOptions
                {
                    Bundling = new BundlingOptions
                    {
                      Image  = Runtime.DOTNET_CORE_3_1.BundlingImage,
                      Command = new []
                      {
                          "bash", "-c", string.Join(" && ", commands)
                      }
                    }
                }),
                Handler = "My.NetCore31.Lambda::My.NetCore31.Lambda.Function::FunctionHandler"
            });
        }
    }
}

```

When using ``Code.fromAsset(path)`` command it is possible to bundle the code found in the "path" folder by running some commands inside a Docker container.    
The "path" will be mounted at ``/asset-input`` and the content at ``/asset-output`` folder will be zipped and used as the Lambda code.

In this particular example when we execute the ``cdk deploy`` command the following steps are going to happen:
- The ``../My.NetCore31.Lambda``  path will be mounted as ``/asset-input`` inside a docker container.
- The commands described in the "Command" attribute will be executed.
  - The first command to be executed will be ``bash -c`` . This command allows us to execute a bunch of joined bash commands. The bash commands to be executed will be:
    - ``cd /asset-input`` command is to step into the folder where our assets are being mounted.
    - The export commands are simply adding into the ``PATH`` environment variable the location of the ``.dotnet/tool`` folder. This is needed because in the next step we're going to install and use a dotnet tool.
    - ``dotnet tool install -g Amazon.Lambda.Tools`` command is installing the dotnet lambda  tool. This tool adds commands to the dotnet cli that can be used manage Lambda functions including deploying a function from the dotnet cli.
    - ``dotnet lambda package -o output.zip`` command is using the global tool to build and publish the lambda function.
    - The last command ``unzip -o -d /asset-output output.zip`` is outputting the result into the ``/asset-output`` folder.
  - The content of the ``/asset-output`` will be zipped and used as the function handler.


# 4. Deploy a NET Core 3.1 lambda using an existing .zip file and the AWS CDK ``FromAsset`` option


> _You can find a working example of this scenario in this [Link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/NetCore31ExistingZipFileLambdaCdk)_

In this scenario the AWS CDK and the application will be deployed at the same time, but you need to have published the lambda function beforehand.



```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;

namespace NetCore31ExistingZipFileLambdaCdk
{
    public class NetCore31LambdaCdkStack : Stack
    {
        internal NetCore31LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {

            Function function = new Function(this,
                "zip-lambda-function",
                new FunctionProps
            {
                Runtime = Runtime.DOTNET_CORE_3_1,
                Code = Code.FromAsset("./src/My.NetCore31.Lambda.zip"),
                Handler = "My.NetCore31.Lambda::My.NetCore31.Lambda.Function::FunctionHandler"

            });
        }
    }
}
```

In the previous scenario we have used the ``Code.fromAsset(path)`` command to create the artifact at deploy time, but the ``Code.fromAsset(path)`` command can also point to a .zip file that contains the published lambda function.

In this particular example when we execute the ``cdk deploy`` command the following steps are going to happen:
- The lambda will be created using the .zip file stored ``./src/My.NetCore31.Lambda.zip`` local folder.