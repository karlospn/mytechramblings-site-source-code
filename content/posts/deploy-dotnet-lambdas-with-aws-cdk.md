---
title: "5 ways to deploy a .NET Lambda using AWS CDK"
date: 2023-02-08T10:01:46+02:00
tags: ["aws", "lambda", "cdk", "dotnet", "docker", "containers"]
description: "This post is going to walk you through 5 different ways to deploy a .NET lambda using AWS CDK."
draft: false
---

> **Show me the code**   
> As always, if you don't care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk)


AWS CDK is a framework that allow us to define cloud infrastructure in code and provision it via CloudFormation.

It offers a high-level object-oriented abstraction to define AWS resources using some of the most well-known programming languages.

Right now, it is available in the following languages:

- JavaScript
- TypeScript 
- Python 
- Java 
- .NET
- Golang

I already shared my thoughts about AWS CDK in some of my older posts, so I'm not gonna bore you talking again about it.   
Instead of that, I will cut right to the chase and focus on showing you some of the available options you have when you want to deploy a .NET Lambda using CDK.

But, before start showing you some code I think that there are 2 concepts worth mentioning:

## Deployment Types

Right now if you want to deploy a Lambda you have **two types of deployments available**: **container images** and **.zip files**.

A container image includes the base operating system, the runtime, Lambda extensions, your application code and its dependencies.   
You upload your container images to Amazon Elastic Container Registry (Amazon ECR).    
To deploy the image to your function, you specify the Amazon ECR image URL using the Lambda console, the Lambda API, command line tools, or the AWS SDKs.

A .zip file archive includes your application code and its dependencies. When you author functions using the Lambda console or a toolkit, Lambda automatically creates a .zip file archive of your code.   
You can upload a .zip file as your deployment package using the Lambda console, AWS Command Line Interface (AWS CLI), or to an Amazon Simple Storage Service (Amazon S3) bucket.

## .NET Managed Runtime

Lambda supports multiple languages through the use of runtimes.   
The runtime provides a language-specific environment that runs in an execution environment. The runtime relays invocation events, context information, and responses between Lambda and the function. You can use managed runtimes that Lambda provides or you can build your own.

The policy with .NET runtimes has always been to support only LTS versions for managed runtimes. So you cannot use .NET 7 as a managed runtime.   

If you want to deploy a .NET 7 lambda you have 2 options available:
- Use the container image deployment type.
- Or if you really want to deploy your lambda as a .zip file you're going to need to use a custom .NET 7 runtime.

Also if you don't mind working with .NET 6 instead of .NET 7 you can use the .zip file deployment straightaway.

---
<br/>

The 5 options to deploy a .NET lambda that I'm going to show you in this post are the following ones:

- **Deploy a .NET 7 lambda using a container image and the AWS CDK ``FromImageAsset`` method.**
- **Deploy a .NET 7 lambda using a container image that has already been pushed into an ECR registry.**
- **Deploy a .NET 6 lambda using a .zip file and the AWS CDK ``FromAsset`` method with bundling options.**
- **Deploy a .NET 6 lambda using an existing .zip file and the AWS CDK ``FromAsset`` method.**
- **Deploy a .NET 6 lambda using a .zip file and the ``XaasKit.CDK.AWS.Lambda.DotNet`` package.**

# **1. Deploy a .NET 7 lambda using a container image and the AWS CDK _"FromImageAsset"_ method**

> _You can find a working example of this scenario in this [link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net7BundlingContainerLambdaCdk)_.

In this scenario the AWS CDK and the lambda function will be deployed at the same time.    

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Constructs;

namespace Net7BundlingContainerLambdaCdk
{
    public class Net7LambdaCdkStack : Stack
    {
        internal Net7LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {
            DockerImageCode dockerImageCode = DockerImageCode.FromImageAsset("../My.Net7.Lambda");

            _ = new DockerImageFunction(this, 
                "container-image-lambda-function",
                new DockerImageFunctionProps()
            {
                Code = dockerImageCode,
                Description = ".NET 7 Docker Lambda function"
            });
        }
    }
}
```

The ``DockerImageCode.FromImageAsset`` method needs to point to a local folder with a 
Dockerfile.   

When we execute the ``cdk deploy`` command the following steps are going to happen:
- The container is going to be built using the dockerfile found in the ``../My.Net7.Lambda`` folder.
- An ECR repository named ``aws-cdk\assets`` will be created if it doesn't exists yet (the **``aws-cdk\assets`` is a centralized repository**, so every lambda created using this method will be pushed into this very same ECR repository).
- The image is going to be pushed into the ``aws-cdk/assets`` repository.
- The lambda will be created amongst the rest of the infrastructure declared in the CDK project.

The next code snippet shows how the dockerfile looks like:
```yaml
FROM public.ecr.aws/lambda/dotnet:7 as base

FROM mcr.microsoft.com/dotnet/sdk:7.0 as build-image
WORKDIR /app

COPY . .
RUN dotnet publish -c Release -o /publish

FROM base AS final
WORKDIR /var/task
COPY --from=build-image /publish .
CMD ["My.Net7.Lambda::My.Net7.Lambda.Function::FunctionHandler"]
```

It's a multi-stage dockerfile:
- The .NET 7 SDK image from Microsoft is used to build and publish the lambda artifact.
- The .NET 7 image for lambdas from AWS is used to run the lambda artifact.

# 2. **Deploy a .NET 7 lambda using a container image that has already been pushed into an ECR registry**

> _You can find a working example of this scenario in this [link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net7ExistingContainerLambdaCdk)._

In this scenario the AWS CDK and the lambda function won't be deployed at the same time. This scenario assumes that you have already pushed the lambda image into an axisting ECR repository.

```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.ECR;
using Amazon.CDK.AWS.Lambda;
using Constructs;

namespace Net7ExistingContainerLambdaCdk
{
    public class Net7LambdaCdkStack : Stack
    {
        internal Net7LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {

            var ecrRepo = Repository.FromRepositoryName(this, 
                "ecr-image-repository", 
                "net7.container.lambda");

            DockerImageCode dockerImageCode = DockerImageCode.FromEcr(ecrRepo);
            
            _ = new DockerImageFunction(this, 
                "container-image-lambda-function", 
                new DockerImageFunctionProps()
            {
                Code = dockerImageCode,
                Description = ".NET 7 Docker Lambda function"
            });
        }
    }
}
```
 
The ``DockerImageCode.FromEcr`` method allows us to specify an image that already exists in ECR as the lambda function handler.

When we execute the ``cdk deploy`` command the following steps are going to happen:
- The lambda will be created using the image stored in the ``net7.container.lambda`` ECR repository.



# **3. Deploy a .NET 6 lambda using a .zip file and the AWS CDK _"FromAsset"_ method with bundling options**

> _You can find a working example of this scenario in this [link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net6BundlingZipFileLambdaCdk)_.

In this scenario the AWS CDK and the lambda function will be deployed at the same time.    

```csharp 
using System.Collections.Generic;
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Constructs;
using AssetOptions = Amazon.CDK.AWS.S3.Assets.AssetOptions;

namespace Net6BundlingZipFileLambdaCdk
{
    public class Net6LambdaCdkStack : Stack
    {
        internal Net6LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {
            IEnumerable<string> commands = new[]
            {
                "cd /asset-input",
                "export XDG_DATA_HOME=\"/tmp/DOTNET_CLI_HOME\"",
                "export DOTNET_CLI_HOME=\"/tmp/DOTNET_CLI_HOME\"",
                "export PATH=\"$PATH:/tmp/DOTNET_CLI_HOME/.dotnet/tools\"",
                "dotnet tool install -g Amazon.Lambda.Tools",
                "dotnet lambda package -o output.zip",
                "unzip -o -d /asset-output output.zip"
            };


            _ = new Function(this,
                "zip-lambda-function",
                new FunctionProps
            {
                Runtime = Runtime.DOTNET_6,
                Code = Code.FromAsset("../My.Net6.Lambda", new AssetOptions
                {
                    Bundling = new BundlingOptions
                    {
                      Image  = Runtime.DOTNET_6.BundlingImage,
                      Command = new []
                      {
                          "bash", "-c", string.Join(" && ", commands)
                      }
                    }
                }),
                Handler = "My.Net6.Lambda::My.Net6.Lambda.Function::FunctionHandler"

            });
        }
    }
}
```

The ``Code.fromAsset(path)`` method allow us to bundle the code found in the "path" folder by running some commands inside a Docker container.    

The "path" will be mounted in the container inside the  ``/asset-input`` folder. Afterwards you can run some commands to bundle the code at runtime.   
The output needs to be stored in the ``/asset-output`` folder.    
Finally the contents of the ``/asset-output/`` folder will be zipped and used as the Lambda handler.

When we execute the ``cdk deploy`` command the following steps are going to happen:
- The ``../My.Net6.Lambda``  path will be mounted as ``/asset-input`` folder inside the docker container.
- The commands described in the "Command" attribute will be executed.
  - The first command to be executed will be ``bash -c`` . This command will allows us to execute series of sequential bash commands. The bash commands to be executed will be:
    - ``cd /asset-input`` command is used to step into the folder where our assets are being mounted.
    - The export commands are simply adding into the ``PATH`` environment variable the location of the ``.dotnet/tool`` folder (this is needed because in the next step we're going to install and use a dotnet tool).
    - ``dotnet tool install -g Amazon.Lambda.Tools`` command is installing the dotnet lambda  tool. This tool adds commands to the dotnet cli that can be used manage Lambda functions including deploying a function from the dotnet cli.
    - ``dotnet lambda package -o output.zip`` command is using the global tool to build, publish and package the lambda function.
    - The last command ``unzip -o -d /asset-output output.zip`` is outputting the result into the ``/asset-output`` folder.
  - The content of the ``/asset-output`` will be zipped and used as the function handler.


# **4. Deploy a .NET 6 lambda using an existing .zip file and the AWS CDK _"FromAsset"_ method**

> _You can find a working example of this scenario in this [link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net6ExistingZipFileLambdaCdk)_.

In this scenario the AWS CDK and the lambda function will be deployed at the same time, but you need to have created the zip file that contains the lambda function beforehand.


```csharp
using Amazon.CDK;
using Amazon.CDK.AWS.Lambda;
using Constructs;

namespace Net6ExistingZipFileLambdaCdk
{
    public class Net6LambdaCdkStack : Stack
    {
        internal Net6LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {

            Function function = new Function(this,
                "zip-lambda-function",
                new FunctionProps
            {
                Runtime = Runtime.DOTNET_6,
                Code = Code.FromAsset("./src/My.Net6.Lambda.zip"),
                Handler = "My.Net6.Lambda::My.Net6.Lambda.Function::FunctionHandler"
            });
        }
    }
}
```

In the previous scenario we have used the ``Code.fromAsset(path)`` command to create the artifact at deploy time, but with the ``Code.fromAsset(path)`` command you can also point to an existing .zip file that contains the published lambda function.

When we execute the ``cdk deploy`` command the following steps are going to happen:
- The lambda will be created using the .zip file stored ``./src/My.Net6.Lambda.zip`` local folder.


# **5. Deploy a .NET 6 lambda using a .zip file and the "XaasKit.CDK.AWS.Lambda.DotNet" package**

> _You can find a working example of this scenario in this [link](https://github.com/karlospn/deploy-dotnet-lambda-with-aws-cdk/tree/main/src/Net6UsingDotNetFunctionLambdaCdk)_.

### **This is experimental.**
### **Right now there is an ongoing RFC proposal discussing the adoption of this implementation.**   
### **If you're interested in trying it out and want to leave some feedback or show some support, go [here](https://github.com/aws/aws-cdk-rfcs/issues/469) and leave a comment.**


One of the main pain points when deploying a lambda function using a .zip file is the build process of the .zip file itself.    
In the last 2 sections, we have seen 2 alternatives for building the artifact .zip file:
- Running a bunch of bash commands using the ``BundlingOptions.Command`` property. _(section 3)_
- Building the artifact beforehand and outside of the CDK and then use the ``Code.FromAsset`` method to point to the .zip file. _(section 4)_

The ``BundlingOptions.Command`` option seems kind of hacky and not reliable at all, and building the artifact beforehand is even worse because then you have to orchestrate more moving parts: create a build pipeline that creates the artifact, place the artifact on an specific folder and then execute the CDK app.

The ``DotNetFunction`` construct has been created for the purpose of streamline this process. This construct can't be found on the official ``Constructs`` NuGet package, if you want to test it by yourself you'll have to install the ``XaasKit.CDK.AWS.Lambda.DotNet`` package.

To use the ``DotNetFunction`` construct you need to have installed both the .NET CLI and the AWS Lambda .NET Tool on your local computer.

```csharp
using Amazon.CDK;
using Constructs;
using XaasKit.CDK.AWS.Lambda.DotNet;

namespace Net6UsingDotNetFunctionLambdaCdk
{
    public class Net6LambdaCdkStack : Stack
    {
        internal Net6LambdaCdkStack(Construct scope, 
            string id,
            IStackProps props = null) : base(scope, id, props)
        {
            _ = new DotNetFunction(this, 
                "my-net6-function", 
                new DotNetFunctionProps
            {
                ProjectDir = "../My.Net6.Lambda",
                Handler = "My.Net6.Lambda::My.Net6.Lambda.Function::FunctionHandler"
            });
        }
    }
}
```
When we execute the ``cdk deploy`` command the following steps are going to happen:
- The AWS Lambda .NET Tool will create the lambda artifact using the source code from the  ``../My.Net6.Lambda`` folder.
- The lambda will be created using the .zip file from the previous step.
