---
title: "Provisioning resources on AWS using AWS CDK and Azure DevOps Pipelines"
date: 2020-09-26T17:04:21+02:00
tags: ["aws","cdk", "azure", "ci", "cd"]
draft: true
---

## Introduction

First of all let me tell you that I'm huge proponent of **Terraform** as a framework for defining infrastructure in code.    
One of the things that I like most about Terraform is that not only every major cloud provider (AWS, Azure, GCP) offers their own provider but each day more and more companies are beginning to offer their own Terraform providers as a viable solution for provisioning resources.   

With Terraform I can create almost any cloud infrastructure that I want and also a varied array of resources such as: VMware vSphere Virtual Machines, RabbitMq Queues, Grafana dashboards amongst many many others.   
And let's not forget that building your own Terraform provider it's pretty easy and it is mainly thanks to a well-thought development framework.

This post is starting to sound like a sales pitch for Terraform... So let's stop talking about Terraform and let's make focus on AWS CDK.

What's AWS CDK and why I was talking about Terraform?   
**AWS CDK** is  another software development framework for defining cloud infrastructure in code.
One of the **main** differences is that it uses AWS CloudFormation for provisioning the resources and basically that means that AWS CDK is a development framework for defining only AWS infrastructure.   

And after talking about all the good things that Terraform can do maybe your asking yourself: "Why should I care about AWW CDK?"    
Well if you're not a multi-cloud / hybrid-cloud enterprise and you're staying entirely within AWS, then AWS CDK is hands-down a much better option than Terraform.    
In my opinion Terraform thrives as a "jack-of-all trades" IaC tool and works great in a multi-cloud / hybrid-cloud environment, meanwhile AWS CDK it has been built to work specifically with AWS and what it does, it does it right.

I'm beginning to drift away again...
What I really wanted to show in this post is how you can deploy AWS CDK applications using Azure DevOps.   



# Step 0 - Initial setup

The main topic of these post is about how to deploy an AWS CDK app using Azure DevOps so I'm not going to talk about how to build an AWS CDK app, also I'm going to assume that those things are already on place:

1 - An **AWS CDK application** containing the resources that we want to deploy.   
The application is going to be stored in an Azure DevOps Git Repository.

> One of the coolest things about AWS CDK is that like Pulumi it uses familiar programming languages (TypeScript, Javascript, Python, Java and .NET) and that's a great point because developers can write with the same language as the rest of their stack.

I'm going to use an already build CDK app from the aws-samples Github Repo. That one: https://github.com/aws-samples/aws-cdk-examples/tree/master/typescript/ecs/fargate-service-with-auto-scaling

2 - A **Service Connection** to your AWS account.   
Obviously that service connection needs to have enough permissions to create the resources you want to provision using CDK.


3 - Having the **_"AWS Toolkit for Azure DevOps_"** extension installed on your Azure DevOps tenant (https://marketplace.visualstudio.com/items?itemName=AmazonWebServices.aws-vsts-tools).   


# Step 1 -  Define the branching and deploy strategy

I'm going for a pretty straight-forward branch strategy.

- The **master** branch contains the **production source code**.
- Every time I want to create a new feature I create a new branch from the master branch.
- The new feature is integrated onto the master branch doing a Pull Request (PR)


![diagram](/img/branching-strategy.png)

We are going to build not one but **2 Azure Pipelines**.

## **Pipeline 1: _pull request validation pipeline_**

When we create a Pull Request from a feature branch into the master branch a validation pipeline is going to be triggered.

That pipeline is going to execute the following steps:

- Validate the CDK output using the cfn-nag (https://github.com/stelligent/cfn_nag)
  - The cfn-nag tool looks for patterns in CloudFormation templates that may indicate insecure infrastructure. It will look for:
    - IAM rules that are too permissive (wildcards)
    - Security group rules that are too permissive (wildcards)
    - Access logs that aren't enabled
    - Encryption that isn't enabled
    - Password literals.   
- Add a comment into the PR specifying with resources are going to be created, modified or deleted.
  - That step is completely **optional** but can be quite useful because the PR reviewer can read the PR comment and know exactly how the infrastructure is going to change once he approves the PR.

## **Pipeline 2: _master branch deployment pipeline_**

When the PR is approved and the code is commited into the master branch and automatic pipeline is triggered.

That pipelines is going to do the following steps:

- Deploy the code.


# Step 2 - Building the PR pipeline

> First of all, let me add a "branch policy" to the master branch so no one can push code without a pull request.

Let me paste the complete PR pipeline here:


```yaml
trigger:
- none

pool:
  vmImage: 'ubuntu-latest'

steps:

- task: NodeTool@0
  inputs:
    versionSpec: '12.x'

- task: UseRubyVersion@0
  inputs:
    versionSpec: '>= 2.4'

- script: |
    echo "Installing packages"
    sudo npm install -g aws-cdk
    sudo gem install cfn-nag
  displayName: 'Installing aws cdk and cfn nag'

- script: |
    echo "Installing project dependencies"
    sudo npm install
    sudo npm run build
  displayName: 'Installing project dependencies'

- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Running validations"
      cdk synth -o out
      cd out
      fname=$(find *.template.json)
      echo "Testing output with cfn-nag-scan"
      cfn_nag_scan --input-path $fname
  displayName: 'Validating AWS CDK output'


- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Generating CDK diff file"
      cdk diff -o out -c aws-cdk:enableDiffNoFail=true --no-color "*" 2>&1 | sed -n '/Resources/,/Outputs/p' | sed 's/├/\+/g; s/│/|/g; s/─/-/g; s/└/`/g' | head -n -1 | tee output.log
      sed -i '1 i\```bash' output.log
      sed -i -e '$a```' output.log
  displayName: 'Generating CDK diff file'

- task: PowerShell@2
  displayName: "Write Pull Request Comment"
  inputs:
    targetType: "inline"
    script: |                      
      try 
      {
    
        Write-Host "Write Pull Request comment"
        $content = Get-Content -Path .\output.log -Raw -ErrorAction Stop
        $organization = "cponsn" 
        $pullRequestThreadUrl = "https://dev.azure.com/$organization/$(System.TeamProjectId)/_apis/git/repositories/$(Build.Repository.ID)/pullRequests/$(System.PullRequest.PullRequestId)/threads?api-version=5.1"
  
        Write-Host "PR Uri: $pullRequestThreadUrl"

        if($Null -eq $content || $content -eq " " || $content -eq "")
        {
            $content = "CDK Diff found no resource is going to change"
        }

      $body = @"
      {
          "comments": [
            {
              "parentCommentId": 0,
              "content": "$content",
              "commentType": 1
            }
          ],
          "status": 4 
      }
      "@
     
        $response = Invoke-RestMethod -Uri $pullRequestThreadUrl -Method Post -Body $body -ContentType "application/json" -Headers @{Authorization = "Bearer $(System.AccessToken)"}
        if ($response -ne $Null) {
          Write-Host "Everything worked"
        }
      }
      catch {
        Write-Error $_
        Write-Error $_.Exception.Message
      }
```

After creating the pipeline we need to add the pipeline as a _"Build Validation"_ in the master branch policy, so the pipeline is triggered when a PR is created.

![build-validation](/img/build-validation.png)

Now, let's try to explain step by step what the pipeline is exactly doing.

1- We begin setting up the _Node_ version for the aws-cdk and the _Ruby_ version for the cfn-nag, after that task we simply install both packages

```yaml
- task: NodeTool@0
  inputs:
    versionSpec: '12.x'

- task: UseRubyVersion@0
  inputs:
    versionSpec: '>= 2.4'

- script: |
    echo "Installing packages"
    sudo npm install -g aws-cdk
    sudo gem install cfn-nag
  displayName: 'Installing aws cdk and cfn nag'

```

2- We are installing the CDK app dependencies and transpiling the code.
If the CDK app was written in csharp instead of Typescript we would be doing something like: _"dotnet build src"_

```yaml
- script: |
    echo "Installing project dependencies"
    sudo npm install
    sudo npm run build
  displayName: 'Installing project dependencies'
```

3- CDK synth creates the CloudFormation template for the specified stack in a concrete folder.   
By default CDK Synth places the Cfn template in the cdk.out folder, but I don't like default behaviours so I'm specifying that I want the Cfn template to be placed in a folder called "out".   
Afterwards I run cfn-nag tool using as a parameter the Cfn template.   

The results of cfn-nag scan are dumped to stdout.
- A failing violation will return a non-zero exit code an stop the pipeline.
- A warning will return a zero/success exit code and the pipelines will continue running.
- A fatal violation stops analysis (per file) because the template is malformed in some severe way.

```yaml
- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Running validations"
      cdk synth -o out
      cd out
      fname=$(find *.template.json)
      echo "Testing output with cfn-nag-scan"
      cfn_nag_scan --input-path $fname
  displayName: 'Validating AWS CDK output'
```

4- CDK diff compares the stack to be deployed with the deployed stack and show the differences between both of them.   

In this step I'm storing the CDK diff output in a file called "output.log" 
Also before saving the output of the CDK diff into the file I'm running some filters using the Linux _sed_ command.   

```yaml
- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Generating CDK diff file"
      cdk diff -o out -c aws-cdk:enableDiffNoFail=true --no-color "*" 2>&1 | sed -n '/Resources/,/Outputs/p' | sed 's/├/\+/g; s/│/|/g; s/─/-/g; s/└/`/g' | head -n -1 | tee output.log
      sed -i '1 i\```bash' output.log
      sed -i -e '$a```' output.log
  displayName: 'Generating CDK diff file'

```

Let me explain a little the purpose of the filters:


```bash
 _sed -n '/Resources/,/Outputs/p'_

 ```

-  In that filter I'm simply removing useless information.   
The CDK diff command shows more information apart from the resources that are going to be created/modified but I only what that piece of information and that's why I'm filtering out everything else.   

  

```bash
sed 's/├/\+/g; s/│/|/g; s/─/-/g; s/└/`/g'
```
- The CDK diff output shows an output similar to the Linux "tree" command so I need to transform the tree output to ASCII, otherwise the PR result will contain missing characters.
That seems to be some kind of Azure DevOps issue, if you're using GitHub instead that step is not necessary.

```bash
  sed -i '1 i\```bash' output.log
  sed -i -e '$a```' output.log
```
Azure DevOps Pull Request comments supports Markdown. So I'm wrapping the output into a markdown code block.  

Let me show you and example at the how the output.log file looks like after applying all the filters. 

```bash
  ```bash
[~] AWS::S3::Bucket MyFirstBucket MyFirstBucketB8884501
 +- [~] DeletionPolicy
 |   +- [-] Retain
 |   `- [+] Delete
 `- [~] UpdateReplacePolicy
     +- [-] Retain
     `- [+] Delete
    ```
```

5- The last step looks a little bit daunting but it's quite simple.   
I'm reading the "output.log" file and using the Azure DevOps API to add a comment in the PR. 

> I'm using Powershell for that step instead of bash because for me it's easier to build a relative "complex" script using Powershell.



```yaml

  displayName: 'Generating CDK diff file'

- task: PowerShell@2
  displayName: "Write Pull Request Comment"
  inputs:
    targetType: "inline"
    script: |                      
      try 
      {
    
        Write-Host "Write Pull Request comment"
        $content = Get-Content -Path .\output.log -Raw -ErrorAction Stop
        $organization = "cponsn" 
        $pullRequestThreadUrl = "https://dev.azure.com/$organization/$(System.TeamProjectId)/_apis/git/repositories/$(Build.Repository.ID)/pullRequests/$(System.PullRequest.PullRequestId)/threads?api-version=5.1"
  
        Write-Host "PR Uri: $pullRequestThreadUrl"

        if($Null -eq $content || $content -eq " " || $content -eq "")
        {
            $content = "CDK Diff found no resource is going to change"
        }

      $body = @"
      {
          "comments": [
            {
              "parentCommentId": 0,
              "content": "$content",
              "commentType": 1
            }
          ],
          "status": 4 
      }
      "@
     
        $response = Invoke-RestMethod -Uri $pullRequestThreadUrl -Method Post -Body $body -ContentType "application/json" -Headers @{Authorization = "Bearer $(System.AccessToken)"}
        if ($response -ne $Null) {
          Write-Host "Everything worked"
        }
      }
      catch {
        Write-Error $_
        Write-Error $_.Exception.Message
      }

```

# Step 4 - Testing the PR pipeline


I have deployed that application : https://github.com/aws-samples/aws-cdk-examples/tree/master/typescript/ecs/fargate-service-with-auto-scaling. To test the pipeline I'm going to modify the CDK app and I will be adding, updating and deleting some resources.

- I'll begin by adding a new S3 bucket into the CDK app and creating a new PR.
  
We wait for the result and the pipeline ends correctly.  
If we drill inside the task where we ran the cfn-nag scan we can see that we have some security warnings that maybe we should try to fix.

![nag-scan](/img/nag-results.png)

And if we take a look at the PR a new comment has been created from the pipeline agent.

![pr-add](/img/pr-add-comment.png)


- Next I'm going to modify the autoscaling policy and the s3 bucket removal policy. After applying the changes in the CDK app I create a new PR.
The pipeline ends correctly and if we take a look at the comment added the PR.

![pr-edit](/img/pr-modification-comment.png)


- Next I'm going to delete the s3 bucket, put the autoscaling policy back to the original value and create a new SNS topic. After applying the changes in the CDK app I create a new PR.
The pipeline ends correctly and if we take a look at PR we can see a new comment has been created.

![pr-delete](/img/pr-delete-comment.png)


- Finally I'm going to tidy a little bit the CDK app and do some code refactoring but I'm not going to modify any resource. After applying some code changes I create a new  PR.
The pipeline ends correctly If we take a look at PR we can see a new comment has been created, but these time is telling us that no resource is going to be modified.

![pr-nochange](/img/pr-nochange-comment.png) 


# Step 5 - Building the master branch integration pipeline

This pipeline is much more simple than the previous one.
The pipeline will be triggered when the PR is completed and the code is commited to the master branch.

```yaml
trigger:
    branches:
        include:
        - master
    paths:
        exclude:
            - 'azure-pipelines-pr.yml'
            - 'azure-pipelines.yml'
pool:
  vmImage: 'ubuntu-latest'

steps:
- script: |
    echo "Installing packages"
    sudo npm install -g aws-cdk
  displayName: 'Installing aws cdk'

- script: |
    echo "Installing project dependencies"
    sudo npm install
    sudo npm run build
  displayName: 'Installing project dependencies'


- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Generating CDK diff file"
      cdk diff -o out -c aws-cdk:enableDiffNoFail=true
  displayName: 'Generating CDK diff'
   

- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Deploying App"
      cdk deploy --ci --require-approval never
  displayName: 'Deploying CDK app'
   
```

Let me explain every step that the pipeline does.

1 - First of all I need to exclude the pipeline files because I don't want to trigger the pipeline if I'm only updating these files.


```yaml
trigger:
    branches:
        include:
        - master
    paths:
        exclude:
            - 'azure-pipelines-pr.yml'
            - 'azure-pipelines.yml'

```

2- We are doing exactly the same steps as the previous pipeline, we're setting up Node and Ruby for the aws-cdk and cfn-nag packages. And afterwards we're installing the app dependencies.


```yaml
- script: |
    echo "Installing packages"
    sudo npm install -g aws-cdk
  displayName: 'Installing aws cdk'

- script: |
    echo "Installing project dependencies"
    sudo npm install
    sudo npm run build
  displayName: 'Installing project dependencies'

```

3 - That step is completely optional, I'm doing it before deploying the code because you might be interested in putting a manual intervention step before deploying the code so the person in charge of the infrastructure can double-check what is going to be created or modified.

```yaml
- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Generating CDK diff file"
      cdk diff -o out -c aws-cdk:enableDiffNoFail=true
  displayName: 'Generating CDK diff'
```

4- Finally we're deploying the app. If the CDK is going to create any security resource like IAM users the pipeline will fail that's why you need to add the flag "--require-approval never"


> If you want to add a manual intervention step before deploying you should create a "Environment" and convert that step into a deployment job. More info here: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops


```yaml
- task: AWSShellScript@1
  inputs:
    awsCredentials: 'AWS DEV ACCOUNT'
    regionName: 'eu-west-1'
    scriptType: 'inline'
    inlineScript: |
      echo "Deploying App"
      cdk deploy --ci --require-approval never
  displayName: 'Deploying CDK app'
   
```

