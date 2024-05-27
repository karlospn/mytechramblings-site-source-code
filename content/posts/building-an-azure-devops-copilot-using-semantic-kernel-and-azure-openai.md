---
title: "Building an Azure DevOps Copilot using .NET 8, Semantic Kernel and Azure OpenAI GPT-4o"
date: 2024-05-26T18:02:37+02:00
description: "This post demonstrates how to create an Azure DevOps Copilot that utilizes a small subset of the Azure DevOps REST API. To achieve this, we will be using Semantic Kernel along with .NET 8 and Azure OpenAI."
tags: ["genai", "azure", "openai", "dotnet", "devops"]
draft: true
---

> **Just show me the code**   
> As always, if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/building-an-azure-devops-copilot-using-semantic-kernel-and-dotnet).

First and foremost, let me clarify that **I don't intend to build a complete Azure DevOps Copilot**, as the Azure DevOps REST API is too big and my spare time is quite limited.

Instead, I plan to create an Azure DevOps Copilot that uses a small subset of the Azure DevOps API.   
My primary goal here is to demonstrate how simple it can be to build your own custom Copilot using Semantic Kernel Plugins when you have a third party API to interact with it.

But what exactly is a Copilot? A Microsoft Copilot is a suite of AI-powered tools integrated into various Microsoft products to assist users in their tasks. It aims to enhance user productivity by providing intelligent, context-aware assistance across a wide range of applications and tasks.     

There are already several examples, such as Microsoft 365 Copilot, GitHub Copilot, and Power Apps Copilot. However, you can also build your own Copilot, and that's exactly what I plan to demonstrate in this post.

In simple terms, we're going to build a chat interface with an AI assistant powered by Azure OpenAI GPT-4o. This assistant will leverage the capabilities of Semantic Kernel plugins to provide answers and perform actions on my Azure DevOps instance, such as managing Team Projects, Git Repositories, Branches, Builds, and more.

But before we start writing code, let's quickly discuss what Semantic Kernel and Semantic Kernel Plugins are.

# **What is Semantic Kernel?**

Semantic Kernel is an open-source SDK designed to facilitate the development of intelligent applications.

It provides a set of tools and libraries that enable developers to build applications capable of understanding, processing, and reasoning about data in a more human-like manner.

Are you familiar with LangChain? Semantic Kernel is an alternative to LangChain. One of its biggest advantages is that, while LangChain is only available in Python and JavaScript, Semantic Kernel is available in .NET.

The fastest way to learn how to use Semantic Kernel is with this C# Notebooks. These notebooks demonstrate how to use Semantic Kernel with code snippets that you can run with a push of a button.

- https://github.com/microsoft/semantic-kernel/blob/main/dotnet/notebooks/README.md

# **Semantic Kernel Plugins**

Imagine you build a chat application that uses Semantic Kernel and GPT. If you try to ask a question related to your specific Azure DevOps instance, what do you think it will respond with? 

Nothing. It won't be able to answer properly because GPT-4o (or any other large language model) doesn't have any knowledge about your Azure DevOps instance.

You can ask general-purpose questions about Azure DevOps, but if you try to ask specific questions about your instance, such as _"How many Team Projects do I have?"_, it will either hallucinate and provide incorrect information or be unable to answer at all.

Semantic Kernel Plugins are a way to add functionality and capabilities to your Copilot. With plugins, you can encapsulate capabilities into a single unit of functionality that can then be run by Semantic Kernel.

They are based on the OpenAI plugin specification and contain both code and prompts. You can use plugins to access data, perform operations or augment your Copilot with any external service. 

![sk-plugins](/img/azdo-copilot-sk-plugins.png)

In this post, we're going to build a Semantic Kernel (SK) plugin that interacts with the Azure DevOps REST API.   
This way, every time we ask a question related to our Azure DevOps instance, Semantic Kernel will use the plugin to interact with Azure DevOps. The result will be sent to GPT-4o, which will then generate a response.

## **What does a SK plugin look like?**

At a high-level, a plugin is a function that can be exposed to SK. The functions within plugins can then be orchestrated by an AI application to accomplish user requests.

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
Forget for a moment about the implementation of the function. What is interesting here is how we're describing everything the function does:
- The purpose of the function: ``[KernelFunction, Description("Read all text from a document")]``
- What it returns: ``[return: Description("Document content")]``
- What are the parameters used for: ``[Description("Path to the file to read")]``

Describing the function and its parameters accurately and precisely is paramount because these description fields are used by Semantic Kernel (SK) when orchestrating functions.   

How would SK know that it needs to run the ``ReadTextAsync`` function when the user asks a question related to its purpose? There are two ways to handle this challenge.

- Invoke the function manually.
- Use the "Auto Function Invocation" feature of Semantic Kernel. This is the approach we want to use. We don't want to run the plugins manually; we want SK to choose the appropriate function for us based on the response from GPT-4.

The auto function invocation feature allows SK to automatically determine which function to invoke based on the context of the user's query and the response from GPT-4. This ensures a seamless and efficient interaction with the Azure DevOps REST API through the SK plugin.

That's enough theory, let's dive into some code.

# **Building the application**

The application we're going to build is a .NET 8 console app that functions as a basic chat client.

Users will be able to ask questions related to their Azure DevOps instance, and the Copilot will utilize Semantic Kernel with our custom Plugin to fetch data from the Azure DevOps instance and respond accordingly.

There are a **few prerequisites** we need before start coding.
- An Azure DevOps instance.
- An Azure OpenAI instance with whatever model you prefer already deployed (I'll be using GPT-4o).


## **1. Building the Chat application**

The first step is to set up Semantic Kernel and build a chat interface, allowing us to ask questions to GPT-4o. Let me show you the complete source code, and then I'll highlight and comment on the most interesting parts.

```csharp
using CustomCopilot.AzureDevOpsPlugin;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;
using Microsoft.SemanticKernel.Plugins.Core;

namespace CustomCopilot
{
    internal class Program
    {
        static async Task Main(string[] args)
        {
            // Create a kernel with the Azure OpenAI chat completion service
            var builder = Kernel.CreateBuilder();
            builder.AddAzureOpenAIChatCompletion(Environment.GetEnvironmentVariable("OAI_MODEL_NAME")!,
                Environment.GetEnvironmentVariable("OAI_ENDPOINT")!,
                Environment.GetEnvironmentVariable("OAI_APIKEY")!);

            // Load the plugins
            #pragma warning disable SKEXP0050
            builder.Plugins.AddFromType<TimePlugin>();
            builder.Plugins.AddFromObject(new AzureDevOpsProjectsPlugin(), nameof(AzureDevOpsProjectsPlugin));
            builder.Plugins.AddFromObject(new AzureDevOpsRepositoriesPlugin(), nameof(AzureDevOpsRepositoriesPlugin));
            builder.Plugins.AddFromObject(new AzureDevOpsBranchesPlugin(), nameof(AzureDevOpsBranchesPlugin));
            builder.Plugins.AddFromObject(new AzureDevOpsCodeSearchPlugin(), nameof(AzureDevOpsCodeSearchPlugin));
            builder.Plugins.AddFromObject(new AzureDevOpsBuildsPlugin(), nameof(AzureDevOpsBuildsPlugin));


            // Build the kernel
            var kernel = builder.Build();

            // Create chat history
            ChatHistory history = [];
            history.AddSystemMessage(@"You are a virtual assistant specifically designed to manage an Azure DevOps instance. Your scope of conversation is strictly limited to this domain. Your responses should be concise, accurate, and directly related to the query at hand.
In order to provide the most accurate responses, you require precise inputs. 
If a function calling involves parameters that you do not have sufficient information about it, it is crucial that you do not attempt to guess or infer their values. Instead, your primary action should always be to ask the user to provide more detailed information about these parameters. This is a non-negotiable aspect of your function. Guessing or inferring values is not an acceptable course of action. Your goal is to avoid any potential misunderstandings and to provide the most accurate and helpful response possible.
Remember, when in doubt, always ask for more information. Never guess the values of a function parameter.
If a function call fails to produce any valid data, the response must always be: 'I'm sorry, but I wasn't able to retrieve any data.If the problem persists, you may want to contact your Azure DevOps administrator or support for further assistance'. 
Never fabricate a response if the function calling fails or returns invalid data.");

            // Get chat completion service
            var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();

            // Start the conversation
            while (true)
            {
                // Trim chat history
                if (history.Count > 20)
                {
                    history.RemoveRange(0, 4);
                }

                // Get user input
                Console.Title = "Azure DevOps Copilot";
                Console.ForegroundColor = ConsoleColor.White;
                Console.Write("\nUser > ");
                history.AddUserMessage(Console.ReadLine()!);

                // Enable auto function calling
                OpenAIPromptExecutionSettings openAiPromptExecutionSettings = new()
                {
                    ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
                };


                // Get the response from the AI
                var response = chatCompletionService.GetStreamingChatMessageContentsAsync(
                               history,
                               executionSettings: openAiPromptExecutionSettings,
                               kernel: kernel);


                Console.ForegroundColor = ConsoleColor.Green;
                Console.Write("\nAssistant > ");

                string combinedResponse = string.Empty;
                await foreach (var message in response)
                {
                    //Write the response to the console
                    Console.Write(message);
                    combinedResponse += message;
                }

                Console.WriteLine();

                // Add the message from the agent to the chat history
                history.AddAssistantMessage(combinedResponse);
            }
        }
    }
}
```

The first step is create and setup Semantic Kernel.    

To add an Azure OpenAI chat completion service to SK, you will need to use the ``AddAzureOpenAIChatCompletion`` method.   
Within the ``AddAzureOpenAIChatCompletion`` method, we're specifing which LLM model we want to use, the Azure OpenAI endpoint, and an Azure OpenAI API Key.

```csharp
// Create a kernel with the Azure OpenAI chat completion service
var builder = Kernel.CreateBuilder();
builder.AddAzureOpenAIChatCompletion(Environment.GetEnvironmentVariable("OAI_MODEL_NAME")!,
    Environment.GetEnvironmentVariable("OAI_ENDPOINT")!,
    Environment.GetEnvironmentVariable("OAI_APIKEY")!);
```

Once we have set up Semantic Kernel (SK) with our Azure OpenAI instance, it's time to add the plugins we want to work with.    

In the next section, we will see how to implement those plugins; for now, let's simply add them to SK.

From the code snippet below, you can see that I'm not going to create a single Azure DevOps plugin. Instead, I'll be creating multiple plugins, each targeting a specific API surface: projects, repositories, branches, builds, etc.   
You can put everything into a single plugin if you prefer; it doesn't change anything functionally. However, I find that segregating the plugins into multiple smaller plugins makes them easier to maintain and update.

Apart from adding our custom Azure DevOps plugins, you can see that we're also adding the ``TimePlugin``. The ``TimePlugin`` comes with Semantic Kernel, so you don't need to implement it. This plugin is used to get time and date information. But why do we need it?

Let me show you an example, and you'll understand it quickly. Imagine we want to ask our Azure DevOps Copilot a time-related question. Let's try it out:

![sk-timeplugin](/img/azdo-copilot-time-plugin-2.png)

From the screen above, you can see that our Copilot, thanks to the ``TimePlugin``, is capable of knowing the current date and can successfully respond to our time-related question.

Now, let's make the same example, but this time, let's NOT add the ``TimePlugin`` to Semantic Kernel.

![sk-no-timeplugin](/img/azdo-copilot-time-plugin-1.png)

Without the ``TimePlugin``, our LLM is incapable of knowing the current date, and therefore its response is incorrect.

```csharp
// Load the plugins
#pragma warning disable SKEXP0050
builder.Plugins.AddFromType<TimePlugin>();
builder.Plugins.AddFromObject(new AzureDevOpsProjectsPlugin(), nameof(AzureDevOpsProjectsPlugin));
builder.Plugins.AddFromObject(new AzureDevOpsRepositoriesPlugin(), nameof(AzureDevOpsRepositoriesPlugin));
builder.Plugins.AddFromObject(new AzureDevOpsBranchesPlugin(), nameof(AzureDevOpsBranchesPlugin));
builder.Plugins.AddFromObject(new AzureDevOpsCodeSearchPlugin(), nameof(AzureDevOpsCodeSearchPlugin));
builder.Plugins.AddFromObject(new AzureDevOpsBuildsPlugin(), nameof(AzureDevOpsBuildsPlugin));
```

Once we have setup everything in SK, it's time to build it.

```csharp
// Build the kernel
var kernel = builder.Build();
```

Next step is to construct the system prompt for our LLM (GPT-4o).    

This is one of the most important parts of the application. It is crucial to properly ground the LLM and prevent it from attempting to guess or infer values when calling our Azure DevOps custom plugin, as it might guess incorrectly. A much better approach is to instruct the LLM to ask the user for the necessary values when in doubt.

Additionally, if the plugin fails to make the call to the Azure DevOps REST API, it is better to instruct the LLM to show an error message instead of attempting to generate a valid response.

```csharp
ChatHistory history = [];
history.AddSystemMessage(@"You are a virtual assistant specifically designed to manage an Azure DevOps instance. Your scope of conversation is strictly limited to this domain. Your responses should be concise, accurate, and directly related to the query at hand.
In order to provide the most accurate responses, you require precise inputs. 
If a function calling involves parameters that you do not have sufficient information about it, it is crucial that you do not attempt to guess or infer their values. Instead, your primary action should always be to ask the user to provide more detailed information about these parameters. This is a non-negotiable aspect of your function. Guessing or inferring values is not an acceptable course of action. Your goal is to avoid any potential misunderstandings and to provide the most accurate and helpful response possible.
Remember, when in doubt, always ask for more information. Never guess the values of a function parameter.
If a function call fails to produce any valid data, the response must always be: 'I'm sorry, but I wasn't able to retrieve any data.If the problem persists, you may want to contact your Azure DevOps administrator or support for further assistance'. 
Never fabricate a response if the function calling fails or returns invalid data.");
```

The final part of the chat application might seem complex, but in reality, we're following the same steps we would take anytime we build a chat with an LLM:
- Get the user's question.
- Send the question to GPT-4o. SK will automatically call our Azure DevOps plugins if the user's question requires it.
- Get the response back and display it.
- Store the response in the chat history.

There are only a couple of things worth mentioning in the code snippet below:
- The chat history is being trimmed to prevent it from growing out of control. There are more sophisticated techniques to manage the ever-growing chat history. The one I'm using is a bit rudimentary (if there are more than 20 messages in the chat history, remove the oldest 4). Nonetheless, it works.
- For SK to automatically call our Azure DevOps plugins when the user's question requires it, we need to set the ``ToolCallBehavior`` property to ``ToolCallBehavior.AutoInvokeKernelFunctions``.

```csharp
// Get chat completion service
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();

// Start the conversation
while (true)
{
    // Trim chat history
    if (history.Count > 20)
    {
        history.RemoveRange(0, 4);
    }

    // Get user input
    Console.Title = "Azure DevOps Copilot";
    Console.ForegroundColor = ConsoleColor.White;
    Console.Write("\nUser > ");
    history.AddUserMessage(Console.ReadLine()!);

    // Enable auto function calling
    OpenAIPromptExecutionSettings openAiPromptExecutionSettings = new()
    {
        ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
    };

    // Get the response from the AI
    var response = chatCompletionService.GetStreamingChatMessageContentsAsync(
                    history,
                    executionSettings: openAiPromptExecutionSettings,
                    kernel: kernel);


    Console.ForegroundColor = ConsoleColor.Green;
    Console.Write("\nAssistant > ");

    string combinedResponse = string.Empty;
    await foreach (var message in response)
    {
        //Write the response to the console
        Console.Write(message);
        combinedResponse += message;
    }

    Console.WriteLine();

    // Add the message from the agent to the chat history
    history.AddAssistantMessage(combinedResponse);
}
```
## **2. Building the Azure DevOps SK Plugins**

In the previous section, we built the Chat Application. Now, it's time to build the Azure DevOps SK Plugins. 

This might sound like a daunting task, but it's actually quite simple:
- Build a C# function that calls the desired endpoint of the Azure DevOps REST API and returns the result.
- Describe the function's purpose using the ``[KernelFunction, Description("...")]`` decorator.
- Describe what the function returns using the ``[return: Description("...")]`` decorator.
- If the function has any parameters, describe what those parameters are for using the ``[Description("...")]`` decorator.

Describing the function and its parameters accurately is paramount because these description fields are used by Semantic Kernel (SK) when deciding if there is any SK Plugin functions that needs to call.  

The next code snippet shows an example of a function responsible for creating a new Azure DevOps Team Project.

```csharp
[KernelFunction, Description("Creates a new Azure DevOps team project if it doesn't exists")]
[return: Description("If the Azure DevOps Team Project creation was succesful or not")]
public async Task<bool> CreateTeamsProject(
    [Description("Name of the new team project")]
    string name,
    [Description("Description of the new team project")]
    string description)
{
    try
    {
        using HttpClient client = new HttpClient();
        client.DefaultRequestHeaders.Accept.Add(
            new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

        client.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Basic",
            Convert.ToBase64String(
                Encoding.ASCII.GetBytes(
                    string.Format("{0}:{1}", "", Environment.GetEnvironmentVariable("AZURE_DEVOPS_PAT")))));

        HttpResponseMessage response = await client.GetAsync(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects?api-version=6.0");

        if (response.IsSuccessStatusCode)
        {
            var responseBody = await response.Content.ReadAsStringAsync();
            dynamic result = JsonConvert.DeserializeObject(responseBody) ?? throw new Exception();

            foreach (var project in result.value)
            {
                if (project.name == name)
                {
                    return false;
                }
            }
        }

        var projectToCreate = new
        {
            name,
            description,
            capabilities = new
            {
                versioncontrol = new { sourceControlType = "Git" },
                processTemplate = new { templateTypeId = "6b724908-ef14-45cf-84f8-768b5384da45" } 
            }
        };

        response = await client.PostAsync(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects?api-version=6.0",
            new StringContent(JsonConvert.SerializeObject(projectToCreate), Encoding.UTF8, "application/json"));

        return response.IsSuccessStatusCode;
    }
    catch (Exception ex)
    { 
        Console.WriteLine(ex.Message);
        return false;
    }
}
```
As you can see from the code snippet above, there is nothing overly complex (the code could be further improved, but I find that it is easier to understand this way).    

The function uses an ``HttpClient`` to fetch the existing Team Projects in my Azure DevOps instance. If the Team Project we want to create already exists, it returns false; otherwise, it makes a second HTTP call to create it and returns 200 Ok.

From this point forward, every functionality we build into our Azure DevOps SK Plugin will follow the same pattern as the one above:
- Describe the function.
- Make some HTTP calls to the Azure DevOps REST API, and return the result.

Therefore, I won't provide extensive commentary from now on. Instead, I'll simply show the code I have implemented and some live examples, so you can see the Copilot in action.

### **Base Class**

As I stated in the previous section, every functionality built into our Azure DevOps SK Plugin follows the same pattern: describe the function, make some HTTP calls to the Azure DevOps REST API, and return the values.

So, I have build a "Base Class" to reduce the duplicated code.

```csharp
protected static HttpClient CreateHttpClient()
{
    var client = new HttpClient();
    client.DefaultRequestHeaders.Accept.Add(new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));
    client.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue(
        "Basic",
        Convert.ToBase64String(Encoding.ASCII.GetBytes($"{string.Empty}:{Environment.GetEnvironmentVariable("AZURE_DEVOPS_PAT")}"))
    );
    return client;
}

protected async Task<dynamic?> GetApiResponse(string requestUri)
{
    using var client = CreateHttpClient();
    var response = await client.GetAsync(requestUri);
    if (response.IsSuccessStatusCode)
    {
        var responseBody = await response.Content.ReadAsStringAsync();
        return JsonConvert.DeserializeObject(responseBody) ?? throw new Exception();
    }
    return null;
}

protected async Task<dynamic?> PostApiResponse(string requestUri, HttpContent content)
{
    using var client = CreateHttpClient();

    var response = await client.PostAsync(requestUri, content);

    if (response.IsSuccessStatusCode)
    {
        var responseBody = await response.Content.ReadAsStringAsync();
        return JsonConvert.DeserializeObject(responseBody) ?? throw new Exception();
    }
    return null;
}
```

### **Azure DevOps Team Project Plugin**

This plugin is responsible for managing the Team Projects on your Azure DevOps instance.

- **Create a new Team Project**

```csharp
[KernelFunction, Description("Creates a new Azure DevOps team project if it doesn't exist")]
[return: Description("If the Azure DevOps Team Project creation was successful or not")]
public async Task<bool> CreateTeamsProject(
    [Description("Name of the new team project")] string name,
    [Description("Description of the new team project")] string description)
{
    try
    {
        var result = await GetApiResponse($"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects?api-version=6.0");
        if (result != null)
        {
            foreach (var project in result.value)
            {
                if (project.name == name)
                {
                    return false;
                }
            }
        }

        var projectToCreate = new
        {
            name,
            description,
            capabilities = new
            {
                versioncontrol = new { sourceControlType = "Git" },
                processTemplate = new { templateTypeId = "6b724908-ef14-45cf-84f8-768b5384da45" }
            }
        };

        using var client = CreateHttpClient();
        var response = await client.PostAsync(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects?api-version=6.0",
            new StringContent(JsonConvert.SerializeObject(projectToCreate), Encoding.UTF8, "application/json")
        );

        return response.IsSuccessStatusCode;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```

- **List all Team Projects**

```csharp
[KernelFunction, Description("Get all existing Azure DevOps team projects")]
[return: Description("A list of names of existing Azure DevOps Team Projects")]
public async Task<List<string>> GetTeamsProject()
{
    try
    {
        var connection = new VssConnection(
            new Uri(Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")!), new VssBasicCredential(string.Empty, Environment.GetEnvironmentVariable("AZURE_DEVOPS_PAT")));
        var projectHttpClient = connection.GetClient<ProjectHttpClient>();
        IPagedList<TeamProjectReference> projects = await projectHttpClient.GetProjects();
        var projectNames = projects.Select(project => project.Name).ToList();
        return projectNames;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return [];
    }
}
```

- **Delete a Team Project**

```csharp
[KernelFunction, Description("Deletes an existing Azure DevOps team project")]
[return: Description("If the Azure DevOps Team Project deletion was successful or not")]
public async Task<bool> DeleteProject(
    [Description("Name of the team project to delete")] string name)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects?api-version=6.0");

        if (result != null)
        {
            foreach (var project in result.value)
            {
                if (project.name == name)
                {
                    using var client = CreateHttpClient();
                    var response = await client.DeleteAsync(
                        $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects/{project.id}?api-version=6.0"
                    );
                    return response.IsSuccessStatusCode;
                }
            }
        }
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```

- **Demo: Using our Copilot to manage the Team Projects**

![sk-plugin-projects](/img/azdo-copilot-projects.png)


### **Azure DevOps Git repositories plugin**

This plugin is responsible for managing the Git repositories on your Azure DevOps instance.    
It is also capable of explaining the purpose of a given Git repository by using the contents of the README file. Additionally, it can provide a list of all the files contained in a repository.

- **List all git repos on a given Team Project**

```csharp
[KernelFunction, Description("Get all existing git repositories on a given team project")]
[return: Description("A list of names of existing git repositories on a given Azure DevOps Team Project")]
public async Task<List<string>> ListGitRepositoriesInProject(
    [Description("Name of the team project")] string projectName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories?api-version=6.0");

        if (result != null)
        {
            var repositories = new List<string>();
            foreach (var repo in result.value)
            {
                repositories.Add((string)repo.name);
            }
            return repositories;
        }
        return [];
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return [];
    }
}
```
- **Create a new git repo**

```csharp
[KernelFunction, Description("Creates a new Git repository, if it doesn't exists, on a given Azure DevOps team project")]
[return: Description("If the git repository creation was successful or not")]
public async Task<bool> CreateGitRepositoryInProject(string projectName, string repositoryName)
{
    try
    {
        using var client = CreateHttpClient();
        
        var projectResponse = await client.GetAsync(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/projects/{projectName}?api-version=6.0");

        if (projectResponse.IsSuccessStatusCode)
        {
            var responseBody = await projectResponse.Content.ReadAsStringAsync();
            var projectDetails = JObject.Parse(responseBody);
            var projectId = projectDetails["id"]?.ToString();

            var repositoryToCreate = new
            {
                name = repositoryName,
                project = new { id = projectId}
            };

            var response = await client.PostAsync(
                $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/git/repositories?api-version=7.1",
                new StringContent(JsonConvert.SerializeObject(repositoryToCreate), Encoding.UTF8, "application/json")
            );
            
            return response.IsSuccessStatusCode;
        }

        return false;

    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```
- **Delete a git repo**

```csharp
[KernelFunction, Description("Deletes an existing git repository from a given Azure DevOps team project")]
[return: Description("If the git repository deletion was successful or not")]
public async Task<bool> DeleteGitRepositoryInProject(
    [Description("Name of the team project to delete")] string projectName,
    [Description("Name of the git repository to delete")] string repositoryName)
{
    try
    {
        var result = await GetApiResponse($"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories?api-version=6.0");
        
        if (result != null)
        {
            foreach (var repo in result.value)
            {
                if (repo.name == repositoryName)
                {
                    using var client = CreateHttpClient();
                    var response = await client.DeleteAsync(
                        $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/_apis/git/repositories/{repo.id}?api-version=6.0"
                    );
                    return response.IsSuccessStatusCode;
                }
            }
        }
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```
- **List all files of a git repo**

```csharp
[KernelFunction, Description("Get all files from an existing git repositories on a given team project")]
[return: Description("A list of all files from an existing git repositories on a given Azure DevOps Team Project")]
public async Task<List<string>> ListFilesInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/items?scopePath=/&recursionLevel=full&api-version=6.0");
        
        if (result != null)
        {
            var files = new List<string>();
            foreach (var item in result.value)
            {
                files.Add((string)item.path);
            }
            return files;
        }
        return [];
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return [];
    }
}
```

- **Explain a git repo** 

It fetches the README file from the main branch and passes it to the LLM, so it can explain the purpose of this Git repository.

```csharp
[KernelFunction, Description("Explain the content of an existing git repositories on a given team project")]
[return: Description("An explanation of what an existing git repositories on a given Azure DevOps Team Project is for")]
public async Task<string> GetReadmeContentInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName)
{
    try
    {
        var client = CreateHttpClient();
        var response = await client.GetAsync(
            $"{
                Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/sourceProviders/tfsGit/fileContents?commitOrBranch=main&repository={repositoryName}&path=/README.md&api-version=6.0-preview.1");

        if (response.IsSuccessStatusCode)
        {
            return await response.Content.ReadAsStringAsync();
        }

        return string.Empty;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return string.Empty;
    }
}
```

- **Demo: Using our Copilot to manage the Git repositories**

**Demo 1**   
![sk-plugin-repos-1](/img/azdo-copilot-repos-1.png)

**Demo 2**  
![sk-plugin-repos-2](/img/azdo-copilot-repos-2.png)

**Demo 3**
![sk-plugin-repos-3](/img/azdo-copilot-repos-3.png)


### **Azure DevOps branches plugin**

This plugin is responsible for managing the branches of a Git repository. It is also capable of providing detailed information about a specific branch.

- **List branches of a git repo**

```csharp
[KernelFunction, Description("Get all branches from a given git repository on a given Azure DevOps team project")]
[return: Description("A list of all branches from an existing git repositories on a given Azure DevOps Team Project")]
public async Task<List<string>> ListBranchesInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?filter=heads&api-version=6.0");

        if (result != null)
        {
            var branches = new List<string>();
            foreach (var branch in result.value)
            {
                branches.Add((string)branch.name);
            }
            return branches;
        }
        return [];
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return [];
    }
}
```

- **Get detailed branch info**

```csharp
[KernelFunction, Description("Get branch info from a given branch from a given git repository on a given Azure DevOps team project")]
[return: Description("Branch details")]
public async Task<string> GetBranchInfoInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName,
    [Description("Name of branch")] string branchName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?filter=heads/{branchName}&api-version=6.0");

        if (result != null && result?.count > 0)
        {
            return JsonConvert.SerializeObject(result!.value[0]);
        }
        return string.Empty;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return string.Empty;
    }
}
```

- **Create a new branch on a given git repo**

```csharp
[KernelFunction, Description("Creates a new branch from a given git repository on a given Azure DevOps team project")]
[return: Description("If the branch creation process was successful or not")]
public async Task<bool> CreateBranchInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName,
    [Description("Name of the source branch that will be used to create a new one")] string sourceBranchName,
    [Description("Name of the new branch that will be created")] string newBranchName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?filter=heads/{sourceBranchName}&api-version=6.0");

        if (result != null && result?.count > 0)
        {
            string oldObjectId = result!.value[0].objectId;
            var content = new[]
            {
                new
                {
                    name = $"refs/heads/{newBranchName}",
                    oldObjectId = "0000000000000000000000000000000000000000",
                    newObjectId = oldObjectId
                }
            };
            using var client = CreateHttpClient();
            var httpContent = new StringContent(JsonConvert.SerializeObject(content), Encoding.UTF8, "application/json");
            var response = await client.PostAsync($"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?api-version=6.0", httpContent);
            return response.IsSuccessStatusCode;
        }
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```

- **Delete a branch**

```csharp
[KernelFunction, Description("Deletes a branch from a given git repository on a given Azure DevOps team project")]
[return: Description("If the branch deletion process was successful or not")]
public async Task<bool> DeleteBranchInGitRepository(
    [Description("Name of the team project")] string projectName,
    [Description("Name of the git repository")] string repositoryName,
    [Description("Name of the branch that will be deleted")] string branchName)
{
    try
    {
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?filter=heads/{branchName}&api-version=6.0");

        if (result != null && result?.count > 0)
        {
            string oldObjectId = result!.value[0].objectId;
            var content = new[]
            {
                new
                {
                    name = $"refs/heads/{branchName}",
                    oldObjectId = oldObjectId,
                    newObjectId = "0000000000000000000000000000000000000000"
                }
            };
            using var client = CreateHttpClient();
            var httpContent = new StringContent(JsonConvert.SerializeObject(content), Encoding.UTF8, "application/json");
            var request = new HttpRequestMessage
            {
                Method = HttpMethod.Post,
                RequestUri = new Uri($"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}/refs?api-version=6.0"),
                Content = httpContent
            };
            var response = await client.SendAsync(request);
            return response.IsSuccessStatusCode;
        }
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return false;
    }
}
```

- **Demo: Using our Copilot to manage branches**

![sk-plugin-branches](/img/azdo-copilot-branches.png)


### **Azure DevOps Builds plugin**

This plugin is responsible for querying builds.

- **Search for builds from a given git repo**

```csharp
[KernelFunction, Description("Search for builds of a given repository on a team project")]
[return: Description("Builds details")]
public async Task<string> GetBuildsInProject(
    [Description("Azure DevOps Team Project")] string projectName,
    [Description("Azure DevOps Git repository")] string repositoryName)
{
    try
    {
        var repoResult = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/git/repositories/{repositoryName}?api-version=6.0");
        if (repoResult == null)
        {
            Console.WriteLine("Failed to get repository ID");
            return string.Empty;
        }

        string repositoryId = repoResult.id;
        var result = await GetApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_URI")}/{projectName}/_apis/build/builds?repositoryId={repositoryId}&repositoryType=TfsGit&api-version=7.2-preview.7");

        if (result != null && result?.count > 0)
        {
            var builds = new List<object>();
            
            foreach (var build in result!.value)
            {
                builds.Add(new
                {
                    build.id,
                    build.buildNumber,
                    build.status,
                    build.result,
                    build.queueTime,
                    build.startTime,
                    build.finishTime,
                    build.sourceBranch,
                    build.sourceVersion,
                    build.url,
                    requestedFor = build.requestedFor.displayName
                });
            }
            return JsonConvert.SerializeObject(builds);
        }
        return string.Empty;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return string.Empty;
    }
}
```

- **Demo: Using our Copilot to query builds**

![sk-plugin-builds](/img/azdo-copilot-builds.png)


### **Azure DevOps CodeSearch plugin**

This plugin is responsible for searching code in a given Team Project.

- **Search text in a given team project**

```csharp
[KernelFunction, Description("Search text in a given team project")]
[return: Description("List of text coincidences")]
public async Task<string> GetCodeSearchInProject(
    [Description("Azure DevOps Team Project")] string projectName,
    [Description("Text Code that must be searched")] string searchText)
{
    try
    {
        var payload = new JObject
        {
            ["searchText"] = searchText,
            ["$skip"] = 0,
            ["$top"] = 50,
            ["filters"] = new JObject
            {
                ["Project"] = new JArray { projectName },
            }
        };

        var content = new StringContent(payload.ToString(), Encoding.UTF8, "application/json");

        var result = await PostApiResponse(
            $"{Environment.GetEnvironmentVariable("AZURE_DEVOPS_ORG_ALM_URI")}/{projectName}/_apis/search/codesearchresults?api-version=7.0", content);
        
        if (result != null && result?.count > 0)
        {
            return JsonConvert.SerializeObject(result);
        }
        return string.Empty;
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex.Message);
        return string.Empty;
    }
}
```

- **Demo: Using our Copilot to search code**

**Demo 1**   
![sk-plugin-cs-1](/img/azdo-copilot-code-search-1.png)

**Demo 2**   
![sk-plugin-cs-2](/img/azdo-copilot-code-search-2.png)

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