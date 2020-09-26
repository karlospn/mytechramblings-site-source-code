---
title: "Provisioning resources on AWS using AWS CDK and Azure DevOps Pipelines"
date: 2020-09-26T17:04:21+02:00
tags: ["aws","cdk", "terraform", "azure"]
draft: true
---

## Introduction


First of all let me tell you that I'm huge proponent of **Terraform** as a framework for defining infrastructure in code.  
One of the things I like the most is that not only every major cloud provider (AWS, Azure, GCP) offers their own Terraform provider but everyday more and more companies are beginning to offer their own Terraform provider as a viable solution for provisioning resources.   
With Terraform I can create almost any cloud infrastructure that I want and also a varied array of resources such as: VMware vSphere Virtual Machines, RabbitMq Queues, Grafana dashboards amongst many many others.   
And let's not forget that building a Terraform provider it's pretty easy and it is mainly thanks to a well-thought development framework. So if you need to create a resource via API you can build pretty quick a Terraform provider that does exactly that.    

Anyways, let's stop talking about Terraform and let's focus on AWS CDK.

What's AWS CDK and why I was talking about Terraform?   
**AWS CDK** is  another software development framework for defining cloud infrastructure in code, but the **main** difference is that it uses AWS CloudFormation for provisioning the resources and basically that means that AWS CDK is a development framework for defining AWS infrastructure.   

And after talking about all the good things that Terraform can do maybe your asking yourself: "Why should I care about it?" If you're not a multi-cloud / hybrid-cloud enterprise and you're staying entirely within AWS, then AWS CDK is hands-down a much better option than Terraform.    
While Terraform works great with AWS, in my opinion Terraform thrives as a "jack-of-all trades" tool and works great in a multi-cloud / hybrid-cloud environment, meanwhile AWS CDK is only for AWS but it has been built to work specifically with AWS and what it does, it does it right.


I'm beginning to drift away again, what I really want to show you in these post is how to deploy AWS CDK applications using Azure DevOps.


# Step 0 - Initial setup


I'm going to assume that those things are already on place:

1 - An **AWS CDK application** containing the resources that we want to deploy. The application is stored in an Azure DevOps Git repository

> One of the coolest things about AWS CDK is that like Pulumi it uses familiar programming languages (TypeScript, Javascript, Python, Java and .NET), you don't need to learn another language and that's great because developers can write with the same code as the rest of their stack.   
I'm not going to talk about how to build an AWS CDK that's a topic for another day (maybe...)

2 - An **Azure DevOps Service Connection** to your AWS account, obviously that service connection needs to have enough permissions to create the resources you have declared on the CDK app.

3 - Having the _"AWS Toolkit for Azure DevOps"_ extension installed on your Azure DevOps tenant (https://marketplace.visualstudio.com/items?itemName=AmazonWebServices.aws-vsts-tools).



# Step 1 -  Building the CI / CD pipeline









