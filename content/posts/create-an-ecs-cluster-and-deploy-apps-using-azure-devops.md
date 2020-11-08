---
title: "How to create an AWS ECS Fargate Cluster and deploy apps using Azure DevOps"
date: 2020-11-08T18:11:18+01:00
draft: true
---


These past couple of weeks I've been tinkering with AWS ECS and after losing some time building different approaches I thought it might be useful to write down what I ended up building.

I'm quite aware that there is already a post on the official AWS blog telling about how you can deploy an app using ECS and Azure DevOps (https://aws.amazon.com/es/blogs/devops/deploying-a-asp-net-core-web-application-to-amazon-ecs-using-an-azure-devops-pipeline/).   

But I'm not really sold on how the blog post does things... let me tell you why:   
The CI/CD pipelines is creating a new image, pushing it to ECR and finally updating the ECS service using the AWS CLI.
First of all I don't want to deploy images using the 'latest' tag strategy. I like to avoid deployments with tags like the latest tag, because those tags continue to receive updates and can introduce inconsistencies in production environments. I prefer to use unique tags for deployments like 'BuildId'.   
The other problem I have is that the pipeline only explains how to update the container, but what about when I don't have a setup? How do I create the ECS Task Definition? How I deploy my application the first time? 
All those questions are linked with one of my **biggest pain points** when working with ECS: 

- **You need an existing image to create an ECS Task Definition**, so I have a problem here..   

And why is this a problem? Because what I like to do is to create the application infrastructure first and then deploy the app. Both things can be deployed using the same pipeline but I don't like to do it on the same step. I like to create the infrastructure first, and the deploy the application and the fact that you need and existing image to create an ECS Task Definition, forces me to follow the following steps:

- Create the ECS cluster.
- Compile the application source code, run the tests, create the image and push it into to ECR repository.
- Create the ECS Task Definition, point the ECS Task Definition to the image I have created in the previous step and finally create or update the ECS Service.

So I end up intermingling the application CI/CD with the creation of the infrastructure.   

Not a fan of that approach, so  I end up building an alternative solution with 3 clear objectives in mind:


# Objectives

- **All the infrastructure in AWS must be created using IaC** (infrastructure-as-code) and must be created using an **Azure DevOps Pipelines**.
- The **application** must be deployed using **another Azure DevOps pipeline**.
- **Decouple the creation, deployment and lifecycle of the infrastructure files from the application source code.**


And the end result look like this:

