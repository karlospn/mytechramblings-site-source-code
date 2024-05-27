---
title: "Building an Azure DevOps Copilot using .NET 8, Semantic Kernel and Azure OpenAi GPT-4o"
date: 2024-05-26T18:02:37+02:00
description: ""
tags: ["genai", "azure", "openai", "dotnet", "devops"]
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/building-an-azure-devops-copilot-using-semantic-kernel-and-dotnet)

First and foremost, let me clarify that **I don't intend to build an entire Azure DevOps Copilot**, as the Azure DevOps REST API is too big and my spare time is quite limited.

Instead, I plan to create an Azure DevOps Copilot that uses a small subset of the Azure DevOps API. My primary goal here is to demonstrate how simple it can be to build your own custom Copilot using Semantic Kernel Plugins when you have a third party API to interact with it.

But what're exactly going to build in here? A Microsoft Copilot is a suite of AI-powered tools integrated into various Microsoft products to assist users in their tasks. It aims too enhance user productivity by providing intelligent, context-aware assistance across a wide range of applications and tasks.

But what exactly are we going to build here? A Microsoft Copilot is a suite of AI-powered tools integrated into various Microsoft products to assist users in their tasks. It aims to enhance user productivity by providing intelligent, context-aware assistance across a wide range of applications and tasks.    
There are already several of them: Microsoft 365 Copilot, GitHub Copilot, Power Platform Copilot, etc. But you can also build your own custom Copilot, and that's exactly what I plan to do.


In plain words, we're going to build a custom Azure DevOps Copilot, which is essentially a chat interface where the AI assistant will be Azure OpenAI GPT-4. This assistant will use the power of Semantic Kernel plugins to perform actions on my Azure DevOps instance, such as managing Team Projects, Git Repositories, Branches, Builds, etc.

But before we start writing code, let's quickly discuss what Semantic Kernel and Semantic Kernel Plugins are.

# **What is Semantic Kernel?**

Semantic Kernel is an open-source SDK designed to facilitate the development of intelligent applications.

It provides a set of tools and libraries that enable developers to build applications capable of understanding, processing, and reasoning about data in a more human-like manner.

Are you familiar with LangChain? Semantic Kernel is an alternative to LangChain. One of the biggest differences is that while LangChain is only available in Python and JavaScript, Semantic Kernel can run in .NET.

The fastest way to learn how to use Semantic Kernel is with this C# and Python Jupyter notebooks. These notebooks demonstrate how to use Semantic Kernel with code snippets that you can run with a push of a button.

- https://github.com/microsoft/semantic-kernel/blob/main/dotnet/notebooks/README.md

# **Semantic Kernel Plugins**

Imagine you build a chat application that uses Semantic Kernel and GPT-4o. If you try to ask a question related to your specific Azure DevOps instance, what do you think will happen? 

It won't be able to answer properly because GPT-4o (or any other LLM) doesn't have any knowledge about your Azure DevOps instance.    
You can ask it general-purpose questions about Azure DevOps, but if you try to ask it specific questions about your instance, such as "How many Team Projects do I have?", it will either hallucinate and provide incorrect information or be unable to answer at all.

Semantic Kernel Plugins are a way to add functionality and capabilities to your Copilot. With plugins, you can encapsulate capabilities into a single unit of functionality that can then be run by Semantic Kernel.

They are based on the OpenAI plugin specification and contain both code and prompts. You can use plugins to access data, perform operations or augment your Copilot with any external service. 

![sk-plugins](/img/azdo-copilot-sk-plugins.png)

In this post, we're going to build a Semantic Kernel (SK) plugin that can interact with the Azure DevOps REST API.   This way, every time we ask a question related to our Azure DevOps instance, Semantic Kernel will use the plugin to interact with Azure DevOps. The result will be sent to GPT-4, and a response will come back.

## **What does a SK plugin look like?**

At a high-level, a plugin is a function that can be exposed to SK. The functions within plugins can then be orchestrated by an AI application to accomplish user request.

The following code is an example of a plugin capable of reading the text of a given document.

```csharp
[KernelFunction, Description("Read all text from a document")]
[return: Description("Document content")]
public async Task<string> ReadTextAsync(
   [Description("Path to the file to read")] string filePath)
{
    using var stream = await this._fileSystemConnector.GetFileContentStreamAsync(filePath).ConfigureAwait(false);
    return this._documentConnector.ReadText(stream);
}
```
Forget about the implementation of the function, what is interested here is that we're describing everything the function does:
- The purpose of the function: ``[KernelFunction, Description("Read all text from a document")]``
- What it returns: ``[return: Description("Document content")]``
- What are the parameters used for: ``[Description("Path to the file to read")]``

Describing the function and its parameters accurately and precisely is paramount because these description fields are used by Semantic Kernel (SK) when orchestrating functions.   

But how would SK know that it needs to run the ``ReadTextAsync`` function when the user asks a question related to its purpose? There are two ways to handle this challenge:

- Invoke the Function manually.
- Use the Auto Function Invocation Feature of Semantic Kernel. This is the approach we want to use. We don't want to run the plugins manually; we want SK to choose the appropriate function for us based on the response from GPT-4.

The auto function invocation feature allows SK to automatically determine which function to invoke based on the context of the user's query and the response from GPT-4. This ensures a seamless and efficient interaction with the Azure DevOps REST API through the SK plugin.

That's enough theory, let's dive into some code.

# **Building the application**

The application we're going to build is a .NET 8 console app that functions as a basic chat client.

Users will be able to ask questions related to their Azure DevOps instance, and the Copilot will utilize Semantic Kernel with our custom Plugins to fetch data from the Azure DevOps instance and respond accordingly.

There are a few prerequisites we need before start coding.
- An Azure DevOps instance.
- An Azure OpenAi instance with whatever model you prefer already deployed (I'll be using GPT-4o).


## **1. Building the Chat application**


## **2. Building the Azure DevOps Semantic Kernel plugins**


# **How to test it**

If you want to test this custom Copilot yourself, you can find the source in my [Github repo](https://github.com/karlospn/building-an-azure-devops-copilot-using-semantic-kernel-and-dotnet).


To run it, you need the following environment variables:

- ``AZURE_DEVOPS_PAT``: A personal access token from your Azure DevOps instance. It is easier if it has full access permissions because we are going to make use of multiple endpoints of the REST API.
- ``AZURE_DEVOPS_ORG_URI``: The URI of your Azure DevOps REST API. The format must be:`` https://dev.azure.com/{your-org}``
- ``AZURE_DEVOPS_ORG_ALT_URI``: The URI of your VSSPS Azure DevOps REST API. The format must be: ``https://vssps.dev.azure.com/{your-org}``
- ``AZURE_DEVOPS_ORG_ALM_URI``: The URI of your ALMSEARCH Azure DevOps REST API. The format must be: ``https://almsearch.dev.azure.com/{your-org}``
- ``OAI_MODEL_NAME``: The LLM name you're going to use. In my case, I'm using ``gpt-4o``. You can use another one of the multiple available models in Azure OpenAI.
- ``OAI_ENDPOINT``: The endpoint of your Azure OpenAI instance. It always has the same format: ``https://{service-name}.openai.azure.com/``
- ``OAI_APIKEY``: An Azure OpenAI Api Key.

Here's an example:
```json
    "AZURE_DEVOPS_PAT": "j093j4194ada123czxsaspdjapsijasfhpi213",
    "AZURE_DEVOPS_ORG_URI": "https://dev.azure.com/cpn",
    "AZURE_DEVOPS_ORG_ALT_URI": "https://vssps.dev.azure.com/cpn",
    "AZURE_DEVOPS_ORG_ALM_URI": "https://almsearch.dev.azure.com/cpn",
    "OAI_MODEL_NAME": "gpt-4o",
    "OAI_ENDPOINT": "https://mytechramblings.openai.azure.com/",
    "OAI_APIKEY": "123123012h032940h213123asdasd"
```