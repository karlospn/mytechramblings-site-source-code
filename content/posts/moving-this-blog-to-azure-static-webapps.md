---
title: "Testing out Azure Static Web Apps service. Let's try to deploy this blog into an static web app"
date: 2020-11-24T10:04:40+01:00
tags: ["azure", "static", "webapps", "github", "action"]
draft: true
---

> If you only care about the end result, I upload the final result in this GitHub repository: **[github-link](https://github.com/karlospn/blog-azure-static-webapp)**   
This repository contains the entire blog source code.   
In the **_"/infrastructure"_** folder you can find an ARM template that is used to create the static web app.   
>In the **/.github/workflows** folder you can find a couple of github actions: 
>   - The **infrastructure.yml** file is a github action that creates the resources on Azure.
>   - The **app-deployment.yml** file is a gitHub action that deploys the blog.


I been wanting to try Azure Static Web Apps for quite some time and I thought that instead of deploying some dummy static site I could use my blog as a test subject.
This blog it's just a bunch of static files, so it should be quite easy to deploy into an Azure Static Web App.   
Right now it uses **Hugo** as a static site framework and **GitHub Pages** to host it.
The current workflow looks like this:

![current-workflow](/img/hugo-blog-workflow.png)

- A GitHub repository hosts the source code. 
- When the code is pushed to master a **github action** builds the source code and pushes the artifact onto another GitHub repository.
- The second GitHub repository is configured as a github page repository and has a custom domain associated.

As you can see it's a super straight-forward CI/CD process: I write a new post, I push it to master and it gets published. No need for anything more complicated.    
So I'm hoping to maintain that simplicity when I move it to Azure Static Web App.

I have 3 things in mind that I want to test:
- **Goal 1**: Use Azure DevOps as my VCS (Version control system) instead of GitHub.
- **Goal 2**: Use an infrastructure-as-code (IaC) approach when creating the resources on Azure.
- **Goal 3**: Maintain the CI/CD simplicity that I have right now on GitHub.


## Goal 1. Moving the blog source code to Azure DevOps

I would like to move the blog source code from GitHub to Azure DevOps because that's where I'm hosting nowadays all my private projects. But it seems that you can't do that.

- **Azure Static WebApps doesn't support Azure DevOps** nowadays. 
- The **only VCS supported right now is GitHub**, you can't use Azure DevOps or GitLab or any other VCS.

After poking around a little bit, it seems that they are working on supporting Azure DevOps in a near future: https://github.com/Azure/static-web-apps/issues/5   

So, I guess my blog is staying on GitHub for now.


## Goal 2. Create the Azure Static Web App blog using IaC

For a project as simple as this I could create the web app directly on the Azure portal, but let's try to follow some good DevOps practices and provision all the resources using IaC.

My de facto tool for IaC is **Terraform**, but it doesn't support Azure Static WebApps right now. It seems that they're also working on it: https://github.com/terraform-providers/terraform-provider-azurerm/pull/7150    
**Pulumi** also doesn't support it, so it seems that my best option right now is using an **ARM template**.   

Here's the ARM template I have built:
```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
        "sku": {
            "type": "string"
        },
        "skucode": {
            "type": "string"
        },
        "repositoryUrl": {
            "type": "string"
        },
        "branch": {
            "type": "string"
        },
        "repositoryToken": {
            "type": "securestring"
        },
        "appLocation": {
            "type": "string"
        },
        "apiLocation": {
            "type": "string"
        },
        "appArtifactLocation": {
            "type": "string"
        },
        "resourceTags": {
            "type": "object"
        }
    },
    "resources": [
        {
            "apiVersion": "2019-12-01-preview",
            "name": "[parameters('name')]",
            "type": "Microsoft.Web/staticSites",
            "location": "[parameters('location')]",
            "tags": "[parameters('resourceTags')]",
            "properties": {
                "repositoryUrl": "[parameters('repositoryUrl')]",
                "branch": "[parameters('branch')]",
                "repositoryToken": "[parameters('repositoryToken')]",
                "buildProperties": {
                    "appLocation": "[parameters('appLocation')]",
                    "apiLocation": "[parameters('apiLocation')]",
                    "appArtifactLocation": "[parameters('appArtifactLocation')]"
                }
            },
            "sku": {
                "Tier": "[parameters('sku')]",
                "Name": "[parameters('skuCode')]"
            }
        }
    ]
}
```
And I'm also using a parameters file:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "value": "mytechramblings-blog"
        },
        "location": {
            "value": "westeurope"
        },
        "sku": {
            "value": "Free"
        },
        "skucode": {
            "value": "Free"
        },
        "repositoryUrl": {
            "value": "https://github.com/karlospn/blog-azure-static-webapp"
        },
        "branch": {
            "value": "master"
        },
        "repositoryToken": {
            "value": "" 
        },
        "appLocation": {
            "value": "/"
        },
        "apiLocation": {
            "value": ""
        },
        "appArtifactLocation": {
            "value": "public"
        },
        "resourceTags": {
            "value": {
                "Environment": "Development",
                "Project": "personal-blog",
                "ApplicationName": "mytechramblings-blog"
            }
        }
    }
}
```

It's a pretty simple ARM template, there is only a couple of things worth mentioning:
- The **"repositoryToken"** parameter must contain a GitHub PAT with admin/write access to the repository and the ability to interact with workflows.    
Right now I'm leaving it as a empty string in the parameters file, and I will inject the correct value during the deployment pipeline.   
- I'm also leaving the **apiLocation** parameter empty, you only need to include an api location if your project uses an Azure Function.


## Goal 3: Building the CI/CD pipeline

Not a big fan of what they have built here, and that's mainly because:

- **An entire CI/CD pipeline is automatically created for you when you create your web app.**
  
After provisioning a new web app if you take a look into your github repository you're going to find out that a **github action has been added**. And that action **already ran once**. 

If you only wanted to create the resources on Azure and deploy the app later you're out of luck, the service is going to deploy the code for you. 
I find that behaviour a little weird, and the worst thing is that I didn't find a way to turn that feature off.   

So without doing absolutely nothing you have a working CI/CD pipeline already built on your GitHub repository. That doesn't sound so bad, right? I don't like it, I want to build the pipeline myself and do whatever I want..

That behaviour trumps my original idea of having a single pipeline that does everything: creates the  infrastructure on Azure, compiles the blog code and finally deploys the artifact into the web app.   
If you try to do that you're going to end up with 2 pipelines that are doing exactly the same: the one that you created and the one that the service has created for you.

So instead of having a single pipeline I guess I should have two of them:

- The first pipeline will create the infrastructure using the ARM template. It will have a manual trigger.

- For the second pipeline I'm going to use the pipeline that the service has created automatically for me.

The resulting workflow will be something like that:

- There is a manual pipeline that creates the resources on Azure.
- When the web app  service itself it will create for me another pipeline that I can use to deploy the blog.
- And from now on I can use the auto-generated pipeline for later deployments. 

Let me show you the github actions:

- **This action creates the Azure static web app using an ARM template**

```yaml
name: Create Azure Static WebApps resources
on:
  workflow_dispatch:
  
env:
  resourceGroupName: 'rg-blog-dev-001'
  resourceGroupRegion: 'westeurope'   
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
    - uses: azure/login@v1
      name: Azure Login
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    - uses: Azure/cli@v1.0.0
      name: Create resource Group 
      with:  
        inlineScript: |
          az group create -l ${{ env.resourceGroupRegion }} -n ${{ env.resourceGroupName }}
    - uses: azure/arm-deploy@v1
      name: Deploy ARM Template
      with:
        subscriptionId: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resourceGroupName: ${{ env.resourceGroupName }}
        template: ./infrastructure/azuredeploy.json
        parameters: ./infrastructure/azuredeploy.parameters.json repositoryToken=${{ secrets.PAT_FOR_AZURE_STATIC_WEBAPPS }}
        deploymentName: mytechramblings-deployment
```


- **This action builds and deploy the blog**.    
I did not build this action it was auto-generated and commited on my repository by the service itself.

```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - master
    paths-ignore: 
      - '.github/**'
      - 'infrastructure/**'
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - master

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v0.0.1-preview
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_PURPLE_FOREST_03D78C803 }}
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for Github integrations (i.e. PR comments)
          action: "upload"
          ###### Repository/Build Configurations - These values can be configured to match you app requirements. ######
          # For more information regarding Static Web App workflow configurations, please visit: https://aka.ms/swaworkflowconfig
          app_location: "/" # App source code path
          api_location: "api" # Api source code path - optional
          output_location: "public" # Built app content directory - optional
          ###### End of Repository/Build Configurations ######

  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v0.0.1-preview
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_PURPLE_FOREST_03D78C803 }}
          action: "close"
```

After executing both pipelines the blog is finally up and running on an static web app.

The last step is to add my custom domain in the web app, you can do it either from the Azure portal or using the Azure CLI:

```bash
az staticwebapp hostname set -n mytechramblings-blog --hostname www.mytechramblings.com
```


## Final thoughts

My impressions after tinkering a little bit with Azure Static WebApps is that it's still lacking a little bit in some departments.

I'm getting the impression that the service is trying a little too hard to automate things for me, it's a nice feature that it creates a fully functional CI/CD pipeline for me, but I want the ability to turn that feature off and do whatever I want to deploy the code.   
Also the support for Azure DevOps is a must. GitHub is great and all, but seems a weird decision to support only GitHub when a huge chunk of .NET shops are using Azure DevOps.

At the end of the day take everything I said here with a grain of salt. The service is still in preview, so it could perfectly be that in a couple a months they have fixed everything I have mentioned here.
