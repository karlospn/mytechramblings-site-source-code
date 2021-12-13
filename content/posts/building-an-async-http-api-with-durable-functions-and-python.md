---
title: "Building an Async HTTP Api with Azure Durable functions and Python"
date: 2021-12-13T13:47:04+01:00
draft: true
tags: ["azure", "functions", "durable", "serverless", "python"]
description: "TBD"
---

> **Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/az-durable-func-async-http-api-python).


In today's post I'll be showing you how to build an Async HTTP API with Azure Durable functions. 

Also, most of the online examples and documentation about Durable Functions are written using csharp so instead of using that I decided to run with Python.   

To begin with, let's do a brief explanation about the key concepts around Azure Durable Functions.

# 1. Azure Durable Functions

For those unaware of Azure Durable Functions, it is an extension of Azure Functions that lets you write stateful functions in a serverless compute environment.   
The extension lets you define stateful workflows by writing orchestrator functions and stateful entities by writing entity functions using the Azure Functions programming model.

The 3 main concepts you should know about are:

- **The Client Function**

The Client Function is responsible for starting stopping and monitoring the orchestrator functions.

- **The Orchestrator Function**

This function is the heart when building a durable function solution. In this function you write your workflow in code. The workflow can consist of code statements calling other functions like activity, or other orchestration functions, or waits for other events to occur or timers.

An orchestration is started by a client function, a function that on its turn can be triggered by a message in a queue, an HTTP request, or any other trigger mechanism you are familiar with. Every instance of an orchestration function will have an instance identifier, which can be auto-generated or user-generated. 

- **The Activity Function**

 These activity functions are the basic unit of work in a durable function solution.    
 Each activity function executes one task and can be anything you want. 


# 2. Problem trying to solve with Azure Durable functions

In this section I'll like to talk a little bit about what I was trying to solve using Durable functions and why Durable functions was a good fit.

One of my clients has an HTTP API that uses an Azure Function as its backend, the function takes a few parameters via querystring, runs a query against an Azure Storage Table and returns a result. It's a really simple setup.

![audit-api](/img/audit-function-durable.png)

But the problem lies in the fact that the query takes between 2 and 7 minutes to complete, so most of the time the HTTP function times out, 230 seconds is the maximum amount of time that an HTTP triggered function can take to respond to a request. This is because of the idle timeout of Azure Load Balancer. 

For longer processing times, we need consider using an async pattern or defer the actual work and return an immediate response.

And Azure Durable Functions provides built-in support for the async http api pattern.


# 3. Async HTTP API Pattern

The async HTTP API pattern addresses the problem of coordinating the state of long-running operations with external clients.  

A common way to implement this pattern is by having an HTTP endpoint trigger the long-running action. Then, redirect the client to a status endpoint that the client polls to learn when the operation is finished.

![async-http-pattern](/img/async-http-api-pattern.png)


# 4. Scenario

![async-http-api](/img/durable-function-async-http-api.png)

# 5. Implementation


# 6. Deploy

# 7. Useful links
- https://docs.microsoft.com/en-us/azure/azure-functions/durable/quickstart-python-vscode
- https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-instance-management?tabs=python
- https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-http-features?tabs=python
- https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp#async-http