---
title: "Trying to move this blog from GitHub Pages to Azure Static Webapps"
date: 2020-11-22T17:14:40+01:00
tags: ["azure", "static", "webapps", "github"]
draft: true
---

I been wanted to test Azure Static WebApps for quite some time, so I have thought that instead of using some dummy static page I'm going to try to migrate this blog to Azure Static WebApps.   

This blog is a very simple one, it's just a bunch of static file where I write wherever and whenever I want. 
It uses **Hugo** as a static site framework and **GitHub Pages** to host it.
The current workflow looks like this:

<FOTO>

- A GitHub repository hosts the source code. 
- When the code is pushed to master a **GitHub Action** builds the source code and pushes the artifact onto another GitHub repository.
- The second GitHub repository is configured as a GitHub Pages repository and have a custom domain associated.

As you can see it's a super simple CI/CD process: I write a new post, I push it to master and it gets published, no need for anything more complicated.    
And I'm hoping to maintain that simplicity when I move it to Azure Static WebApps.

I have 3 goals in mind that I want to achieve with this move to Azure Static WebApps:
- Goal 1: Move the blog source to my Azure DevOps tenant.
- Goal 2: Use an infrastructure-as-code (IaC) approach when creating the resources on Azure.
- Goal 3: Maintain the CI/CD simplicity that I have right now on GitHub.


## Goal 1. Moving the blog source code to Azure DevOps

One of my main goal when I have started with this migration is moving the source code from GitHub to Azure DevOps because that's where I'm hosting nowadays all my private projects. But here is where I found the first problem:

- **Azure Static WebApps doesn't support Azure DevOps** nowadays. In fact you can **ONLY host and deploy your code from GitHub**, what a weird decision...   

But to be fair Azure Static WebApps it's still on preview and after poking around the  Azure GitHub repository it seems that they are working on it: https://github.com/Azure/static-web-apps/issues/5

So, I guess I'm staying on GitHub for now...


## Goal 2. Create the Azure Static WebApps blog using IaC

I'm going a little bit fancy here. For a  project as simple as this I could create the WebApps directly from the Azure portal, but let's try to follow some good DevOps practices and provision all the resources using IaC.

My de facto IaC tool is Terraform, but Terraform doesn't support Azure Static WebApps, it seems they're working on it: https://github.com/terraform-providers/terraform-provider-azurerm/pull/7150

Pulumi also doesn't support it, so it seems that my best option right now is using ARM templates.   

Let me show you the final result. That's the ARM template:

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
And that's the parameters file:

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
            "value": "https://github.com/karlospn/dotnetramblings_source"
        },
        "branch": {
            "value": "feature/static-web-app"
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
                "Environment": "Dev",
                "Project": "personal-blog",
                "ApplicationName": "mytechramblings-blog"
            }
        }
    }
}
```
The **"repositoryToken"** parameter must contain a GitHub PAT with admin/write access to the repository and the ability to interact with workflows.    
Right now I'm leaving it empty and I will inject the right value during the CI/CD pipeline.   

I'm also leaving the apiLocation parameter empty. You should include an api location only if your project uses an Azure Function.

And finally I'm pushing these ARM template files into a folder called **"infrastructure"** onto my blog git repository.

<FOTO>

## Goal 3: Building the CI/CD pipeline

That's another decision that I'm not a fan of it.   

- When you create a new static WebApps on Azure **the service creates for you the CI/CD pipeline on your GitHub repository**.   
So after provisioing the webapp on Azure you're going to find out that a Github workflow has been pushed into your repository. And also has already ran once.   
And the worst thing is that it seems that this behaviour can't be turn off...

The good thing about that behaviour is that without doing absolutely nothing you have an automatic CI/CD pipeline already built for you.

The bad thing is that you can't build the CI/CD pipeline as you desire. The service is going to create a pipeline and ran it and it seems that you can't do nothing about it. After the pipeline has ran once you can delete it or modify it, but still don't like it.

That behaviour trumps my original idea of having a single pipeline that does everything: creates the needed infrastructure on Azure, compiles the blog source code and deploys it.   
You can't do that because as I said because you're going to end up with 2 pipelines that are doing the same: the one that you created and the one that the service has created for you.

So I decided to split the single pipeline that I had in mind and instead build 2 pipelines:
- The first one is tackle the creation of the infrastructure and it will have a manual trigger.
- The second pipeline is going to compile the blog source code and deploy it. For this pipeline I'm going to modify a little bit the one that the service has created automatically for me and with some minimal modification I'm good to go.

Let me show you the end result:

- Pipeline 1:

```yaml

```

- Pipeline 2:

```yaml

```