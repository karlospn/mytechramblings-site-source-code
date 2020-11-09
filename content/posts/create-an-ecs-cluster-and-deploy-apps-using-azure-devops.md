---
title: "An opinionated approach about how to create an AWS ECS Fargate cluster and deploy apps on it using Azure DevOps Pipelines"
date: 2020-11-08T18:11:18+01:00
tags: ["aws", "ecr", "ecs", "containers", "devops", "cdk"]
draft: true
---

These past couple of weeks I've been tinkering with AWS ECS Fargate and after losing some time tackling different approaches I thought it might be useful to write down what I ended up building.

First of all I tried to emulate this post on the official AWS blog: https://aws.amazon.com/es/blogs/devops/deploying-a-asp-net-core-web-application-to-amazon-ecs-using-an-azure-devops-pipeline/

My first thought was: "Lucky me, this post is exactly what I'm trying to build", but in the end didn't like the approach.  Let me explain a little bit my problems with it: 

1- I didn't want to deploy images using the 'latest' tag strategy. 

To deploy a container into ECS you need to create a ECS Task Definition, and this Task Definition contains among many other attributes the the Uri and tag of the image so if you put something like 'uri:latest' you don't need to create a new Task Definition everytime you deploy a version of the same container.   
That's a neat trick, but I really like to avoid deployments with stable tags like the 'latest' tag, because those tags continue to receive updates and can introduce inconsistencies in production environments. I prefer to use unique tags for every deployment like the container build Id.   

2- The other problem I had is that the pipeline only explains how to update the container, but how should I deploy my application the first time? How I create the ECS Task Definition and the ECS Service? And those questions are linked with one of my **biggest pain points** when working with ECS:

- **You need an existing image to create an ECS Task Definition**   

And why is it a problem? Because what I like to do is to create the application infrastructure first and then deploy the app. Both things can be deployed using the same pipeline but I don't like to do it at the same time.   
I like to create the infrastructure first and then deploy the application, and the fact that you need and existing image to create an ECS Task Definition, forces me to do something along these lines:

- Create the ECS Fargate cluster and the Application Load Balancer (ALB).
- Compile the application source code, run the tests, create the image and push it into an ECR repository.
- Create the ECS Task Definition and point it to the image I have pushed in the previous step.
- Create an ALB Target Group that will contain the ECS tasks.
- Create the ECS Service.

So in my mind it just seems that I'm intermingling the creation of the infrastructure with the application CI/CD.

In my perfect world I would rather do something like this:

- Create the ECS Fargate cluster
- Create the ALB
- Create the ECS Task Definition
- Create the ALB target Group
- Create the ECS Service

And after all the infrastructure required to run my application is created then deploy the application onto that infrastructure.  
And that's exactly what I end up building, I'm sure that some people might not agree with that approach and find the other one completely fine, but nonetheless it worked for me. 
When I started building that solution I sat down and write 3 clear objectives I wanted to achieve.

- **All the infrastructure in AWS must be created using IaC** (infrastructure-as-code) and must be created using an **Azure DevOps Pipelines**.
- The **application** must be deployed using **another Azure DevOps pipeline**.
- **Decouple the creation, deployment and lifecycle of the infrastructure files from the application source code.**

I ended up building 3 different separate pipelines:
- A pipeline to create a dummy application.
- A pipeline to create the infrastructure to AWS
- A pipeline to deploy the application.   
  
Let me explain a little bit about the purpose of every pipeline and how everything works together.

# 1. Create and deploy a _"dummy"_ application 

As I have stated before you need an existing image when creating an ECS Task Definition so I decided to create a "dummy" application.   
That dummy application is going to be used when provisioning the infrastructure, so when all my infrastructure is provisioned I'll have an ECS cluster running a "hello-world" application. 

And afterwards when the application CI/CD pipeline kicks in is going to replace the dummy application with the real one.

That's the first pipeline:

![push-dummy-image-to-ecr](/img/push-dummy-image-to-ecr.png)

It's just a very simple pipeline: build the container image and push it into an ECR repository.

- That pipeline will only **run ONCE** and that's it.    
  
And why is it going to run just once? Because I'm using that image as a stepping stone so I can create all my infrastructure without needing to reference a real container image.   
Once the image is pushed into a generic ECR repository I can reuse it for all the applications that I decide to spun up inside my ECS cluster. 

So that's the first pipeline you are going to ran but you can ran it whenever you want as long as it is the first one.

As I said it's a very simple pipeline, take a look:

```yaml
trigger:
  branches:
    include:
    - master
  paths:
    include:
    - '*'
    exclude:
    - 'azure-pipelines.yml'

pool:
  vmImage: 'ubuntu-latest'

variables:
  - name: AWS_ACCOUNT_ID
    value: 1234567789927
  - name: APP_NAME
    value: dummy
  - name: DOCKERFILE_PATH
    value: Dummy.WebApi/Dockerfile
  - name: AWS_REGION
    value : eu-west-1
  - name: TAG
    value: $(Build.BuildId)

stages:
- stage: 'BuildAndPushToECR'
  jobs:
  - job: 'Build'
    steps:    
    - script: |
        echo Starting docker build
        echo docker build -t $(APP_NAME):$(TAG) -f $(System.DefaultWorkingDirectory)/$(DOCKERFILE_PATH) .
        docker build -t $(APP_NAME):$(TAG) -f $(System.DefaultWorkingDirectory)/$(DOCKERFILE_PATH) .
      displayName: 'Run docker build' 

   - task: ECRPushImage@1
      displayName:  Push image to Dev ECR
      inputs:
        awsCredentials: 'AWS ECR Dev'
        regionName: '$(AWS_REGION)'
        imageSource: 'imagename'
        sourceImageName: '$(APP_NAME)'
        sourceImageTag: '$(TAG)'
        repositoryName: '$(APP_NAME)'
        pushTag: '$(TAG)'

```

# 2. Create and deploy the infrastructure needed on AWS

I decided to use AWS CDK to create the infrastructure.   
I'm not going to show the CDK code that I have developed, because it's a pretty straightforward one. The only thing worth mentioning is that the ECS Task Definition is created referencing the _"dummy"_ image I have built in the previous step. 

```csharp
    private const string DUMMY_IMAGE = "arn:aws:ecr:eu-west-1:1234567789927:repository/dummy";

    private FargateTaskDefinition CreateTaskDefinition(
        EcsServiceConstructProps props, 
        ILogGroup logGroup)
    {

        var task = new FargateTaskDefinition(this,
            "td-app",
            new FargateTaskDefinitionProps
            {
                Cpu = props.Cpu,
                TaskRole = props.TaskDefinitionRole,
                ExecutionRole = props.TaskExecutionRole,
                Family = props.FamilyName,
                MemoryLimitMiB = props.Memory
            });

        task.AddContainer("td-app-container", 
            new ContainerDefinitionOptions
        {
            Cpu = props.Cpu,
            MemoryLimitMiB = props.Memory,
            Image = ContainerImage.FromEcrRepository(Repository.FromRepositoryArn(this, 
                "dummy-image", DUMMY_IMAGE)),
            Logging = LogDriver.AwsLogs(new AwsLogDriverProps
            {
                StreamPrefix = "ecs",
                LogGroup = logGroup
            }),
            Environment = GetEnvironmentVariables(props)
        }).AddPortMappings(new PortMapping
        {
            ContainerPort = props.ContainerPort,
        });

        return task;
    }

```

The pipeline is quite simple as I'm just executing the CDK app. 

![cdk-deploy](/img/cdk-deploy.png)

I'm not going to show you the pipeline to deploy a CDK app because I already talked about it in a previous post.   
If you want to know how to build an Azure DevOps pipeline that deploys an AWS CDK app you can read my previous post: https://www.mytechramblings.com/posts/provisioning-aws-resources-using-cdk-azure-devops/


# 3. Deploy the application

The last pipeline is the one in charge of building and deploying the application.

![ecs-update](/img/ecs-update.png)

- The "Continuous Integration" (CI) creates the docker image, uses the Build Id variable to tag it and finally  pushes the image into the AWS ECR repository.   
- The "Continous Deployment" (CD) creates a new revision of the ECS Task Definition that points to the image we have just uploaded and updates the ECS Service to use newly created Task Definition.

The tricky point of this pipeline is when trying to create a new task definition and update the service. We can do it by ourselves via scripting and the result will be something like this:

```bash
#!/bin/bash

set -e
ECR_IMAGE="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"

TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition "$TASK_FAMILY" --region "$AWS_DEFAULT_REGION")

NEW_TASK_DEFINTIION=$(echo $TASK_DEFINITION | jq --arg IMAGE "$ECR_IMAGE" '.taskDefinition | .containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities)')

NEW_TASK_INFO=$(aws ecs register-task-definition --region "$AWS_DEFAULT_REGION" --cli-input-json "$NEW_TASK_DEFINTIION")

NEW_REVISION=$(echo $NEW_TASK_INFO | jq '.taskDefinition.revision')
aws ecs update-service --cluster ${ECS_CLUSTER} \
                       --service ${SERVICE_NAME} \
                       --task-definition ${TASK_FAMILY}:${NEW_REVISION}```

```

And you just need to add an AWS Shell Task in your pipeline (https://docs.aws.amazon.com/vsts/latest/userguide/awsshell.html) with the former script or we can use the Fargate CLI : https://github.com/awslabs/fargatecli   

The fargate CLI is an unofficial command-line interface but nonetheless it allows me to streamline the deployment of containers to an existing AWS Fargate cluster.

You can update the Task Definition image attribute and restart the ECS Service simply by running the following command:

```bash
fargate service  deploy ${ECS_SERVICE_NAME} --image ${ECR_REPOSITORY_ARN}:${Build.BuildId}--cluster ${CLUSTER_ARN} --region ${AWS_DEFAULT_REGION}
```

For simplicity I decided to ran with the Fargate CLI instead of the bash script, I place the CLI inside Azure Devops on a separate Git repository.   
This pipeline looks a little more complex, so let me show the end result:

```yaml
trigger:
  branches:
    include:
    - master
  paths:
    include:
    - '*'
    exclude:
    - 'azure-pipelines.yml'

pool:
  vmImage: 'ubuntu-latest'

resources:
  repositories:
  - repository: FargateCliRepository
    type: git
    name: 'DevOpsTools/ECSToolset'
    ref: master

variables:
  - name: AWS_ACCOUNT_ID
    value: 1234567789927
  - name: APP_NAME
    value: chat-app
  - name: DOCKERFILE_PATH
    value: Chat.WebApi/Dockerfile
  - name: AWS_REGION
    value : eu-west-1
  - name: FARGATE_CLUSTER_ARN
    value: arn:aws:ecs:$(AWS_REGION):$(AWS_ACCOUNT_ID):cluster/product-dev-cluster
  - name: SERVICE_NAME
    value: chat-app-dev-ecs-svc   
  - name: ECR_REPOSITORY
    value: $(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com/$(APP_NAME)
  - name: FARGATE_CLI_PATH
    value: ECSToolset/fargate-cli/linux64
  - name: TAG
    value: $(Build.BuildId)

stages:
- stage: 'BuildAndPushToECR'
  jobs:
  - job: 'Build'
    steps:    
    - checkout: self  
    - script: |
        echo Starting docker build
        echo  docker build -t $(APP_NAME):$(TAG) -f $(System.DefaultWorkingDirectory)/$(DOCKERFILE_PATH) .
        docker build -t $(APP_NAME):$(TAG) -f $(System.DefaultWorkingDirectory)/$(DOCKERFILE_PATH) .
      displayName: 'Run docker build'  
   - task: ECRPushImage@1
      displayName:  Push image to ECR
      inputs:
        awsCredentials: 'AWS ECR Dev'
        regionName: '$(AWS_REGION)'
        imageSource: 'imagename'
        sourceImageName: '$(APP_NAME)'
        sourceImageTag: '$(TAG)'
        repositoryName: '$(APP_NAME)'
        pushTag: '$(TAG)'
- stage : 'DeployToECSFargateCluster'
  jobs:
  - job:      
    steps:  
      - checkout: FargateCliRepository
      - checkout: self     
      - task: AWSShellScript@1
        inputs:
          awsCredentials: 'AWS Fargate Dev'
          regionName: '$(AWS_REGION)'
          scriptType: 'inline'
          inlineScript: |
            cd $(System.DefaultWorkingDirectory)/$(FARGATE_CLI_PATH)
            ./fargate service deploy $(SERVICE_NAME) --image $(ECR_REPOSITORY):$(TAG) --cluster $(FARGATE_CLUSTER_ARN) --region $(AWS_REGION)
        displayName: 'Update Task Definition'

```

As I showed you I ended up with 3 pipelines instead of a single pipeline that does everything, for me personally that's more comfortable because these pipelines are going to be maintained by different people, but if you want you can put together these 3 pipelines in a single one. 