---
title: "How to convert a few .NET apps into .NET templates, package them in a single NuGet and use them within Visual Studio. Part 2: Creating the MyTechRamblings.Templates package"
date: 2021-06-10T14:25:29+02:00
draft: true
tags: ["dotnet", "csharp", "templates", "vs", "visual", "studio"]
---

> **Show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/pack-dotnet-templates-example).   
Also I have upload the NuGet package to [nuget.org](https://www.nuget.org/packages/MyTechRamblings.Templates).

# **Creating the MyTechRamblings.Templates package**

I'll be creating 2 solution templates a also a project template, and I will try to use them within Visual Studio.   
You need to remember what I said in part 1:

> - **Using .NET CLI templates in Visual Studio is only available in Visual Studio version 16.8 or higher**.    
> - **Also if you're creating a solution template you need at least Visual Studio version 16.10 or higher.**

## 1. Prerequisites:

**I have develop 3 apps beforehand**, that's because in this post I want to focus on the process of converting these 3 apps in templates, package them in a NuGet and use it within Visual Studio.

The 3 apps I have built are the following ones:
- **NET 5 Web Api**
  - It is an entire solution app. It uses a N-layer architecture with 3 layers:
    - ``WebApi`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The api has the following features already built-in:
    - ``Azure Pipelines`` YAML file.
    - ``GitHub Action`` file.
    - ``HealthChecks``
    - ``Swagger``
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Identity.Web``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` that allow us to deploy the api into an Azure App Service.

  >If you want to take a look at the source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/WebApiNet5Template)

- **NET 5 Worker Service that consumes RabbitMq messages**
  - It is an entire solution app. It creates a ``BackgroundService`` that consumes messages from a RabbitMq server. It uses a N-layer architecture with 3 layers:
    - ``WebApi`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The service has the following features already built-in:
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Extensions.Hosting.Systemd``
    - ``Microsoft.Extensions.Hosting.WindowsServices``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action``that allow us to deploy the service into an Azure App Service.

  >If you want to take a look at the source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/HostedServiceNet5RabbitConsumerTemplate)

- **NET Core 3.1 Azure Function that gets triggered by a timer**
  - This one is not an entire solution, instead  it is a single project. 
  - It creates a NET Core 3.1 Azure Function that is triggered by a timer.
  - The function has the following features already built-in:
    - HealthChecks
    - Swagger
    - Serilog
    - AutoMapper
    - Microsoft.Identity.Web
  - The function also has the following features already built-in:
    - ``Dependency Injection``
    - ``Logging``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` that allow us to deploy the function.

  >If you want to take a look at the source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/AzureFunctionTimerProjectTemplate)

---

In the following sections I will be converting these 3 apps into templates, package them in a  NuGet file named ``MyTechRamblings.Templates`` and showing you how to use them within Visual Studio.

## 2. Convert the NET 5 Web Api into a template

### 2.1. **Create the template.json file**

The first step will be to create the ``template.json`` file, but before start building it you need to know what you want to parameterize in the template.

After taking a look at the different features I have built on the api, I have come with a list of features that I want to parameterize:

| Parameter             | Description                                                                                                                                                                                                                                                                                                | Default value           |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------|
| **Docker**                | Adds a Dockerfile image.                                                                                                                                                                                                                                   | _true_                    |
| **ReadMe**                | Adds a README markdown file describing the project.                                                                                                                                                                                                                                                     | _true_                    |
| **Tests**                 | Adds an Integration Test project and a Unit Test project in the solution.                                                                                                                                                                                                                                                 | _true_                    |
| **GitHub**                | Adds a GitHub Action file that allow us to deploy the api into an Azure Web App.                                                                                                                                                                                                                          | _false_                   |
| **AzurePipelines**        | Adds an Azure pipeline YAML file that allow us to deploy the api into an Azure Web App.                                                                                                                                                                                                                   | _true_                    |
| **DeploymentType**        | Specifies how you want to deploy the api. The possible values are "DeployAsZip" or "DeployAsContainer".    Depending of the value you choose it will create a different deployment pipeline.    If you choose to not create nor a GitHub Action neither an Azure Pipeline this parameter is useless. | _DeployAsZip_             |
| **AcrName**               | An Azure ACR registry name. Only used if you are going to be deploying with container.                                                                                                                                                                                                                     | _acrcponsndev_            |
| **AzureSubscriptionName** | An Azure DevOps Service Endpoint subscription. Only used if you are going to be deploying with Azure Pipelines.                                                                                                                                                                                            | _cponsn-dev-subscription_ |
| **AppServiceName**        | The name of the Azure App Service where the code will be deployed.                                                                                                                                                                                                                                         | _app-svc-demo-dev_        |
| **Authorization**         | Enables the use of authorization using Microsoft.Identity.Web                                                                                                                                                                                                                                              | _true_                    |
| **AzureAdTenantId**       | Azure Active Directory Tenant Id. Only necessary if Authorization is enabled.                                                                                                                                                                                                                              | _8a0671e2-3a30-4d30-9cb9-ad709b9c744a_                       |
| **AzureAdDomain**         | Azure Active Directory Domain Name. Only necessary if Authorization is enabled.                                                                                                                                                                                                                            | _carlosponsnoutlook.onmicrosoft.com_                       |
| **AzureAdClientId**       | Azure Active Directory App Client Id. Only necessary if Authorization is enabled.                                                                                                                                                                                                                          | _fdada45d-8827-466f-82a5-179724a3c268_                       |
| **AzureAdSecret**         | Azure Active Directory App Secret Value. Only necessary if Authorization is enabled.                                                                                                                                                                                                                       | _1234_                       |
| **HealthCheck**           | Enables the use of healthchecks.                                                                                                                                                                                                                                                                           | true                    |
| **HealthCheckPath**       | HealthCheck path. Only necessary if HealthCheck is enabled.                                                                                                                                                                                                                                                | _/health_                 |
| **Swagger**               | Enables the use of Swagger.                                                                                                                                                                                                                                                                                | _true_                    |
| **SwaggerPath**           | Swagger path. Do not add a backslash. Only necessary if Swagger is enabled.                                                                                                                                                                                                                                | _api-docs_                |
| **Contact**               | The contact details to use if someone wants to contact you. Only necessary if Swagger is enabled.                                                                                                                                                                                                          | _user@example.com_        |
| **CompanyName**           | The name of the company. Only necessary if Swagger is enabled.                                                                                                                                                                                                                                             | _mytechramblings_         |
| **CompanyWebsite**        | The website of the comany. Only necessary if Swagger is enabled.                                                                                                                                                                                                                                           | _www.mytechramblings.com_ |
| **ApiDescription**        | The description of the api. Only necessary if Swagger is enabled.                                                                                                                                                                                                                                          | _Put your api info here._  |




- If you are working with Docker just set the ``Docker`` parameter to true and a dockerfile will be placed alongside your api.
- If you want to add some tests in you solution set the ``Tests`` parameter to true. A unit test project and a integration test project will be placed in your ``/test`` folder.
- If the api is going to be deployed with Azure Devops set the ``AzurePipelines`` attribute to true. A YAML pipeline and a ``/pipelines`` folder will be created inside your solution. 
- If the api is going to be deployed with GitHub set the ``GitHub`` attribute to true. A YAML file containing a GitHub Action and a ``/.github`` folder will be created inside your solution.
- Depending of how the api is going to be deployed set the ``DeploymentType`` attribute accordingly. This attribute will change the pipelines accordingly.
  - If you deploy using containers it will give you a container deployment pipeline.
  - If you deploy using a zip file it will give you an artifact deployment pipeline.
- You can enable or disable features like ``HealthChecks``or ``Swagger``.
- If you are using Authorization with Azure Active Directory, set the ``Authorization`` attribute to true and set the ``AzureAdTenantId``, ``AzureAdDomain``, ``AzureAdClientId``, ``AzureAdSecret`` attributes accordingly. 

There is a default value for each attribute. You can put a default value that matches with your most common scenario so you won't need to set all those values every time you want to create a new app.

After listing which features I want to parameterize, here's the ``template.json`` file:

```javascript
{
    "$schema": "http://json.schemastore.org/template",
    "author": "mytechramblings.com",
    "classifications": [
      "Cloud",
      "Service",
      "Web"
    ],
    "name": "NET 5 WebApi",
    "description": "A WebApi solution.",
    "groupIdentity": "Dotnet.Custom.WebApi",
    "identity": "Dotnet.Custom.WebApi.CSharp",
    "shortName": "mtr-api",
    "defaultName": "WebApi1",
    "tags": {
      "language": "C#",
      "type": "solution"
    },
    "sourceName": "ApplicationName",
    "preferNameDirectory": true,
    "primaryOutputs": [
        { "path": "ApplicationName.sln" }
    ],
    "sources": [
        {
          "modifiers": [
                {
                    "condition": "(!Docker)",
                    "exclude":
                    [
                        "Dockerfile"
                    ]
                },
                {
                    "condition": "(!ReadMe)",
                    "exclude": 
                    [
                        "README.md"
                    ]
                },
                {
                    "condition": "(!Tests)",
                    "exclude": 
                    [
                        "test/ApplicationName.Library.Impl.UnitTest/**/*",
                        "test/ApplicationName.WebApi.IntegrationTest/**/*"
                    ]
                },
                {
                    "condition": "(!GitHub)",
                    "exclude": 
                    [
                        ".github/**/*"
                    ]
                },
                {
                    "condition": "(!AzurePipelines)",
                    "exclude": 
                    [
                        "pipelines/**/*"
                    ]
                },
                {
                    "condition": "(!Swagger)",
                    "exclude": 
                    [
                      "src/ApplicationName.WebApi/Extensions/ServiceCollectionExtensions/ServiceCollectionSwaggerExtension.cs"
                    ]
                },
                {
                    "condition": "(!HealthCheck)",
                    "exclude": 
                    [
                      "src/ApplicationName.WebApi/Extensions/ServiceCollectionExtensions/ServiceCollectionHealthChecksExtension.cs"
                    ]
                }
            ]
        }
    ],
    "symbols": {
        "Docker": {
            "type": "parameter",
            "datatype": "bool",
            "description": "Adds an optimised Dockerfile to add the ability to build a Docker image.",
            "defaultValue": "true"
        },
        "ReadMe": {
            "type": "parameter",
            "datatype": "bool",
            "defaultValue": "true",
            "description": "Add a README.md markdown file describing the project."
        },
        "Tests": {
            "type": "parameter",
            "datatype": "bool",
            "defaultValue": "true",
            "description": "Adds an integration and unit test projects."
        },
        "GitHub": {
            "type": "parameter",
            "datatype": "bool",
            "description": "Adds a GitHub Actions continuous integration pipeline.",
            "defaultValue": "false"
        },
        "AzurePipelines": {
            "type": "parameter",
            "datatype": "bool",
            "description": "Adds an Azure Pipelines YAML.",
            "defaultValue": "true"
        },
        "DeploymentType": {
            "type": "parameter",
            "datatype": "choice",
            "choices": [
              {
                "choice": "DeployAsContainer",
                "description": "The app will be deployed as a container."
              },
              {
                "choice": "DeployAsZip",
                "description": "The app will be deployed as a zip file."
              }
            ],
            "defaultValue": "DeployAsZip",
            "description": "Select how you want to deploy the application."
          },
        "DeployContainer": {
            "type": "computed",
            "value": "(DeploymentType == \"DeployAsContainer\")"
        },
        "DeployZip": {
            "type": "computed",
            "value": "(DeploymentType == \"DeployAsZip\")"
        },
      "AcrName": {
          "type": "parameter",
          "datatype": "string",
          "defaultValue": "acrcponsndev",
          "replaces": "ACR-REGISTRY-NAME",
          "description": "An Azure ACR registry name. Only used if deploying with containers."
      },
      "AzureSubscriptionName": {
          "type": "parameter",
          "datatype": "string",
          "defaultValue": "cponsn-dev-subscription",
          "replaces": "AZURE-SUBSCRIPTION-ENDPOINT-NAME",
          "description": "An Azure Subscription Name. Only used if you are going to be deploying with Azure Pipelines. "
      },
      "AppServiceName": {
          "type": "parameter",
          "datatype": "string",
          "defaultValue": "app-svc-demo-dev",
          "replaces": "APP-SERVICE-NAME",
          "description": "An Azure App Service Name."
      },
      "Authorization": {
        "type": "parameter",
        "datatype": "bool",
        "defaultValue": "true",
        "description": "Enables the use of authorization with Microsoft.Identity.Web."
      },
      "AzureAdTenantId":{
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "8a0671e2-3a30-4d30-9cb9-ad709b9c744a",
        "replaces": "AAD-TENANT-ID",
        "description": "Azure Active Directory Tenant Id. Only necessary if Authorization is enabled."
      },
      "AzureAdDomain":{
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "carlosponsnoutlook.onmicrosoft.com",
        "replaces": "AAD-DOMAIN",
        "description": "Azure Active Directory Domain Name. Only necessary if Authorization is enabled."
      },
      "AzureAdClientId":{
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "fdada45d-8827-466f-82a5-179724a3c268",
        "replaces": "AAD-CLIENT-ID",
        "description": "Azure Active Directory App Client Id. Only necessary if Authorization is enabled."
      },
      "AzureAdSecret":{
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "1234",
        "replaces": "AAD-SECRET-VALUE",
        "description": "Azure Active Directory App Secret Value. Only necessary if Authorization is enabled."
      },
      "HealthCheck": {
        "type": "parameter",
        "datatype": "bool",
        "defaultValue": "true",
        "description": "Enables the use of healthchecks."
      },
      "HealthCheckPath": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "/health",
        "replaces": "HEALTHCHECK-PATH",
        "description": "HealthCheck path. Only necessary if HealthCheck is enabled."
      },
      "Swagger": {
        "type": "parameter",
        "datatype": "bool",
        "defaultValue": "true",
        "description": "Enable the use of Swagger."
      },
      "SwaggerPath": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "api-docs",
        "replaces": "SWAGGER-PATH",
        "description": "Swagger UI Path. Do not add a backslash. Only necessary if Swagger is enabled."
      },
      "Contact": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "user@example.com",
        "replaces": "API-CONTACT",
        "description": "The contact details to use if someone wants to contact you. Only necessary if Swagger is enabled."
      },
      "CompanyName": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "mytechramblings",
        "replaces": "COMPANY-NAME",
        "description": "The name of the company. Only necessary if Swagger is enabled."
      },
      "CompanyWebsite": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "www.mytechramblings.com",
        "replaces": "COMPANY-WEBSITE",
        "description": "The website of the company. Only necessary if Swagger is enabled."
      },
      "ApiDescription": {
        "type": "parameter",
        "datatype": "string",
        "defaultValue": "Put your api info here",
        "replaces": "API-DESCRIPTION",
        "description": "The description of the WebAPI. Only necessary if Swagger is enabled."
      }
    },
    "SpecialCustomOperations": {
        "**/*.yml": {
            "operations": [
              {
                "type": "conditional",
                "configuration": {
                  "if": [ "#if" ],
                  "else": [ "#else" ],
                  "elseif": [ "#elseif" ],
                  "endif": [ "#endif" ],
                  "actionableIf": [ "##if" ],
                  "actionableElse": [ "##else" ],
                  "actionableElseif": [ "##elseif" ],
                  "actions": [ "uncomment", "reduceComment" ],
                  "trim": "true",
                  "wholeLine": "true",
                  "evaluator": "C++"
                }
              },
              {
                "type": "replacement",
                "configuration": {
                  "original": "#",
                  "replacement": "",
                  "id": "uncomment"
                }
              },
              {
                "type": "replacement",
                "configuration": {
                  "original": "##",
                  "replacement": "#",
                  "id": "reduceComment"
                }
              }
            ]
        }
    }
}
```

Let make an in-depth rundown of what's the meaning of every value:

- ``author``: The author of the template.
- ``classifications``: Zero or more characteristics of the template that a user might search for it by. This classifications will appear as the template tags in the .NET CLI and in VS.
- ``name``: The name of the template. That's the name that will appear in the .NET CLI and in VS.
- ``description``:  The description of the template. The description will appear in the .NET CLI when you ran the ``dotnet new <TEMPLATE_NAME> -h`` command. In VS it will appear in the "Create new project dialog".
- ``groupIdentity``: The ID of the group this template belongs to.
- ``identity``: A unique name for this template.
- ``shortName``: A short name used for selecting the template in the .NET CLI. This is the name that shows when you run ``dotnet new -l`` and is the one that you will use when you want to create a new solution using the template. It has no use in Visual Studio.
- ``defaultName``: The name that will be used when you create a new solution using the template if no name has been specified. 
- ``tags``: You can add multiple tags but at least you have to specify the ``language`` tag and the ``type`` tag.
  - The language tag specifies the programming language. 
  - The type tag specifies the type of the template project. The possible values are: project, solution, item. 
- ``sourceName``: The template engine will look for any occurrence of the ``sourceName`` and replace it. It will rename files it there is an occurrence. It will also replace file content if there is an occurrence.   
As you can see I'm using _"ApplicationName"_ as the sourceName. Also  I'm naming every csproj with the "ApplicationName" prefix:
  - ApplicationName.sln
  - ApplicationName.WebApi.csproj
  - ApplicationName.Library.Contracts.csproj
  - ApplicationName.Library.Impl.csproj
  - ApplicationName.Repository.Contracts.csproj
  - ApplicationName.Repository.Impl.csproj   

  When you create a new solution using the template you will be asked to specify the "sourceName" for this new solution and every csproj and sln will be renamed with the name chosen by the user.

- ``preferNameDirectory``: Indicates whether to create a directory for the template. If you set the value to ``false`` the content will be created in the current directory.

- ``primaryOutputs``: TEST

- ``specialCustomOperations``: The templating engine only supports conditional operators in a certain list of file types. If you need to add conditionals operators in another file types you need to add it here.    
In my template I want to add conditional operators on the YAML files, so that's why I'm adding an custom operation on any yml file.

- ``sources.modifiers`` : The sources.modifiers allows us to include or exclude files based on a condition.

- ``symbols``: The symbols section is where you specify the template inputs and define what those inputs should do. 

In our case the ``symbols`` section and the ``source.modifiers`` section are intertwined.
- I'm defining symbol in the ``symbol`` section and the value from the symbol is used as a condition in the ``source.modifier`` section.

The ``symbols`` and the ``sources.modifiers`` section is the meat of the ``template.json`` file, so let's enumerate the different scenarios you can find.

### **Symbol replacement**

- Replaces a fixed string with the value of the symbol.

**Example:** 

Here I have the symbol ``HealthCheckPath`` of type ``parameter``. It has a default value of ``/health`` and a replace value of ``HEALTHCHECK-VALUE``. 
```javascript
  "HealthCheckPath": {
    "type": "parameter",
    "datatype": "string",
    "defaultValue": "/health",
    "replaces": "HEALTHCHECK-PATH",
    "description": "HealthCheck path. Only necessary if HealthCheck is enabled."
  }
```
When you try to create a new solution using the template:

- The template engine will try to find the "HEALTHCHECK-PATH" string anywhere in the template and if it finds it, it will be replaced either by the user input or the default value (/health) 
If you take a look in the ``Startup.cs``, you'll see the ``replaces`` value hard-coded in the template.

```csharp
   endpoints.MapHealthChecks("HEALTHCHECK-PATH", new HealthCheckOptions
    {
      ResponseWriter = ApplicationBuilderWriteResponseExtension.WriteResponse
    });
```

When the user creates a new solution using the template the magic string ``HEALTHCHECK-PATH`` will be replaced by the user input.

If you take a look at the ``template.json`` file a lot of symbols are using this concrete approach.

### **Use conditional operators based on the symbol value**

- It allows us to skip entire chunks of code based on a symbol value.

**Example:**
Here I have the symbol ``Authorization`` of type ``bool`` with the default value of ``true``.

```javascript
  "Authorization": {
    "type": "parameter",
    "datatype": "bool",
    "defaultValue": "true",
    "description": "Enables the use of authorization with Microsoft.Identity.Web."
  },
```

- If the user sets the value to ``false``, the template engine needs to remove any ``Microsoft.Web.Identity`` reference from the solution created.
- If the user sets this symbol to ``true``, the template engine needs to keep the references to the ``Microsoft.Web.Identity``.

If you take a look at the  ``Startup.cs``, you'll see that all the ``Microsoft.Web.Identity`` references are being wrapped in a conditional.

```csharp
#if Authorization
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Identity.Web;
#endif

...

#if Authorization            
            services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddMicrosoftIdentityWebApi(Configuration, "AzureAd");
#endif

#if Authorization
            app.UseAuthentication();
            app.UseAuthorization();
#endif
```

We are keeping the ``Microsoft.Web.Identity`` code only if the symbol is set to ``true``. But that's not enough, we need to remove the ``Microsoft.Web.Identity`` references from the ``appsettings.json`` file too.

```javascript
{
  //#if (Authorization)  
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "AAD-TENANT-ID",
    "Domain": "AAD-DOMAIN",
    "ClientId": "AAD-CLIENT-ID",
    "ClientSecret": "AAD-SECRET-VALUE",
    "TokenValidationParameters": {
      "ValidateIssuer": true,
      "ValidIssuer": "https://login.microsoftonline.com/AAD-TENANT-ID/v2.0",
      "ValidateAudience": true,
      "ValidAudiences": [ "AAD-CLIENT-ID" ]
    }
  }
  //#endif
}
```

And also from the api controller:

```csharp
#if Authorization
    [Authorize]
#endif
    [ApiVersion("1.0")]
    [ApiController]
    [ProducesResponseType(typeof(ErrorResult),  StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ErrorResult),  StatusCodes.Status500InternalServerError)]
    [Route("v{version:apiVersion}/[controller]")]
    [Produces("application/json")]
    public class MyServiceController : ControllerBase
    {
    ...
    }

```

## Use a symbol value to compute another one

- Given a value of a symbol, you can set the value of another one.

**Example:**

The ``DeploymentType`` symbol is of type ``choice`` and with its value you can set the ``DeployContainer`` symbol value and the ``DeployZip`` symbol value.

```javascript
 "DeploymentType": {
    "type": "parameter",
    "datatype": "choice",
    "choices": [
      {
        "choice": "DeployAsContainer",
        "description": "The app will be deployed as a container."
      },
      {
        "choice": "DeployAsZip",
        "description": "The app will be deployed as a zip file."
      }
    ],
    "defaultValue": "DeployAsZip",
    "description": "Select how you want to deploy the application."
  },
  "DeployContainer": {
    "type": "computed",
    "value": "(DeploymentType == \"DeployAsContainer\")"
  },
  "DeployZip": {
    "type": "computed",
    "value": "(DeploymentType == \"DeployAsZip\")"
  },
```

The ``DeployContainer`` symbol and  the ``DeployZip`` symbol will be used to tailor the deployment pipeline. 
- If the symbol ``DeploymentContainer`` is set to ``true``, the deployment pipeline will build and deploy a docker image.
- If the symbol ``DeployZip`` is set to ``true``, the deployment pipeline will build and deploy zipped artifact.   

This is how the Azure Pipelines YAML file looks like:

```yaml
trigger:
- master

pool: 
  vmImage: 'ubuntu-latest'

#if (DeployContainer)
variables:
  - name: registryName
    value: 'ACR-REGISTRY-NAME'
  - name: azureSubscription
    value: 'AZURE-SUBSCRIPTION-ENDPOINT-NAME'
  - name: appServiceName
    value: 'APP-SERVICE-NAME'

steps:
  - task: AzureCLI@2
    displayName: AZ ACR Login
    inputs:
      azureSubscription: $(azureSubscription)
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: 'az acr login --name $(registryName)'

  - task: AzureCLI@2
    displayName: AZ ACR Build
    inputs:
      azureSubscription: $(azureSubscription)
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: 'az acr build -t ApplicationName:$(Build.BuildId) -t ApplicationName:latest -r $(registryName) -f Dockerfile .'
      useGlobalConfig: true
      workingDirectory: '$(Build.SourcesDirectory)'

  - task: AzureWebAppContainer@1
    displayName: Deploy to App Service
    inputs:
      azureSubscription: '$(azureSubscription)'
      appName: '$(appServiceName)'
      containers: '$(registryName).azurecr.io/$(imageName):latest'
#endif

#if (DeployZip)
variables:
  - name: buildConfiguration
    value: 'Release'
  - name: azureSubscription
    value: 'AZURE-SUBSCRIPTION-ENDPOINT-NAME'
  - name: appServiceName
    value: 'APP-SERVICE-NAME'

steps:
- task: DotNetCoreCLI@2
  displayName: Restore
  inputs:
    command: 'restore'
    projects: '**/*.csproj'

- task: DotNetCoreCLI@2
  displayName: Build
  inputs:
    command: 'build'
    projects: '**/*.csproj'
    arguments: '--configuration $(buildConfiguration) --no-restore'

- task: DotNetCoreCLI@2
  displayName: Test
  inputs:
    command: 'test'
    projects: '**/*UnitTest.csproj'
    arguments: '--configuration $(buildConfiguration) --no-restore'

- task: DotNetCoreCLI@2
  displayName: Publish
  inputs:
    command: publish
    publishWebProjects: True
    arguments: '--configuration $(BuildConfiguration) --output $(build.artifactstagingdirectory)'
    zipAfterPublish: True

- task: AzureRmWebAppDeployment@4
  inputs:
    ConnectionType: 'AzureRM'
    azureSubscription: '$(azureSubscription)'
    appType: 'webApp'
    WebAppName: '$(appServiceName)'
    packageForLinux: '$(build.artifactstagingdirectory/*.zip'
#endif
```

## Excluding files based on a symbol value:

- You can combine the ``source.modifiers`` section with the ``symbols`` section to remove existing files based on a symbol value.

**Example 1:**

The ``Dockerfile`` symbol is of type ``parameter`` and the default value is ``true``.

```javascript
  "Docker": {
      "type": "parameter",
      "datatype": "bool",
      "description": "Adds an optimised Dockerfile to add the ability to build a Docker image.",
      "defaultValue": "true"
  }
```

- If the symbol value is set to ``false``, the template engine needs to remove the "Dockerfile" file from the template.
- If the symbol value is set to ``true``, the template engine needs to keep the "Dockerfile" file.

To achieve this behaviour, you can add the following object in ``source.modifiers`` array.

```javascript
  {
      "condition": "(!Docker)",
      "exclude":
      [
          "Dockerfile"
      ]
  }
```

The template engine will remove the "Dockerfile" file if the symbol "Docker" equals to false.


**Example 2:**

This is a more interesting example. The ``Test`` symbol is of type ``parameter`` and the default value is ``true``

```javascript
  "Tests": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "true",
      "description": "Adds an integration and unit test projects."
  }
```

- If the symbol value is set to ``false``, the template engine needs to remove the entire folder that contains the unit test project and also the folder that contains the integration test. It also needs to remove the references from the solution file.
- If the symbol value is set to ``true``, the template engines needs to keep the both test projects.

To achieve this behaviour, you can add the following object in ``source.modifiers`` array.

```javascript
  {
      "condition": "(!Tests)",
      "exclude": 
      [
          "test/ApplicationName.Library.Impl.UnitTest/**/*",
          "test/ApplicationName.WebApi.IntegrationTest/**/*"
      ]
  },
```

The template engine also needs to exclude the test project references from the ``.sln`` file.  To do it we can use a conditional check.

```csharp
#if (Tests)
Project("{2150E333-8FDC-42A3-9474-1A3956D46DE8}") = "Test", "Test", "{0323DDF8-0F80-4DF1-8F12-C37353534337}"
EndProject
#endif
...
#if (Tests)
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "ApplicationName.WebApi.IntegrationTest", "test\ApplicationName.WebApi.IntegrationTest\ApplicationName.WebApi.IntegrationTest.csproj", "{205B0B75-C24F-40E4-BA69-46AE61CF9C20}"
EndProject
Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "ApplicationName.Library.Impl.UnitTest", "test\ApplicationName.Library.Impl.UnitTest\ApplicationName.Library.Impl.UnitTest.csproj", "{C22AA50D-81CC-4A9B-9926-7AB8445246A9}"
EndProject
#endif
```
### 2.2. **Create the dotnetcli.host.json file**

### 2.3. **Create the ide.host.json file**

## 3. Convert the remaining 2 apps into templates

- We still need to convert the ``Worker Service`` application  and the ``Azure Function`` into a .NET template.

But the conversion process looks exactly the same.   
The ``template.json`` file is almost identical with few different parameters, so I'm not going to bother writing about it. 
If you're interested in the end result, take a look at the GitHub repository.

## 4. Create the MyTechRamblings.Templates NuGet package

In **section 1.2** I described a couple of ways to package a template NuGet. 

I'm using a ``.nuspec`` file at the root level. It looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package>
  <metadata>
    <id>MyTechRamblings.Templates</id>
    <version>0.1.0</version>
    <description>This nuget is an example about how to pack multiple dotnet templates in a single NuGet package and use it within Visual Studio or the .NET CLI.</description>
    <authors>Carlos Pons (www.mytechramblings.com)</authors>
    <title>MyTechRamblings Dotnet Templates</title>
    <copyright>Copyright 2021: www.mytechramblings.com. All right Reserved</copyright>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <license type="expression">MIT</license>
    <tags>.NET WebApi Rabbit Function Templates</tags>
    <projectUrl>https://github.com/karlospn/pack-dotnet-templates-example</projectUrl>
    <repository type="git" url="https://github.com/karlospn/pack-dotnet-templates-example.git" branch="main"  />
    <language>en-US</language>
	<packageTypes>
      <packageType name="Template" />
    </packageTypes>
  </metadata>
  <files>
    <file src="**" exclude="**\bin\**\*;**\obj\**\*;**\*.user;**\*.lock.json;_rels\**\*;package\**\*" />
  </files>
</package>
```

A template pack, in the form of a .nupkg NuGet package, requires that **all templates be stored in the content folder** within the package.    
So I have created a powershell that does exactly this and also bundles everything together using the ``nuget.exe`` executable.

```powershell
Set-StrictMode -Version Latest
$templateName = "template"
$templatePath =     "./$templateName/mtr"
$contentDirectory = "./$templateName/mtr/content"
$nugetPath = "./$templateName/nuget.exe"
$nugetOut =  "./$templateName/nuget"
$nugetUrl = "https://dist.nuget.org/win-x86-commandline/v5.9.1/nuget.exe"


Write-Output "Copy WebApiNet5 template"
Copy-Item -Path "./src/WebApiNet5Template" -Recurse -Destination "$contentDirectory/WebApiNet5Template" -Container

Write-Output "Copy HostedServiceNet5RabbitConsumer template"
Copy-Item -Path "./src/HostedServiceNet5RabbitConsumerTemplate" -Recurse -Destination "$contentDirectory/HostedServiceNet5RabbitConsumerTemplate" -Container

Write-Output "Copy AzureFunctionTimerProjectTemplate template"
Copy-Item -Path "./src/AzureFunctionTimerProjectTemplate" -Recurse -Destination "$contentDirectory/AzureFunctionTimerProjectTemplate" -Container

Write-Output "Copy nuspec"
Copy-item -Force -Recurse "MyTechRamblings.Templates.nuspec" -Destination $templatePath

Write-Output "Download nuget.exe from $nugetUrl"
Invoke-WebRequest -Uri $nugetUrl -OutFile $nugetPath

Write-Output "Pack nuget"
$cmdArgList = @( "pack", "$templatePath\MyTechRamblings.Templates.nuspec",
				 "-OutputDirectory", "$nugetOut", "-NoDefaultExcludes")
& $nugetPath $cmdArgList 
```
As you can see it's a quite simple script.
- Move the 3 templates inside a ``/content`` folder.
- Download nuget.exe and run ``nuget.exe pack`` command using the ``.nuspec`` file.


## 5. Install and use the MyTechRamblings.Templates with the .NET CLI


To Install the package you can run this command:

-  ``dotnet new -i MyTechRamblings.Templates::0.1.0``   

It will try to fetch the template NuGet from nuget.org and install it. 

Or if you have the nupkg in your local directory, just ran this other one:

-  ``dotnet new -i .\MyTechRamblings.Templates.0.1.0.nupkg``

After installing the templates pack you should run the ``dotnet new -l`` command and verify that the 3 templates appear on the list.

![dotnet-list-templates](/img/dotnet-list-installed-templates.png)

Now you can create a new solution using the api templates just running:

-  ``dotnet new mtr-api``

This command will create a solution with all the default values. If you want to specify a concrete value just execute the ``dotnet new mtr-api -h`` command to list all the possible options.

Or you can create a new solution using the worker service template running:

- ``dotnet new mtr-rabbit-worker``

Or you can create an Azure Function project using the azure function template running:

- ``dotnet new mtr-az-func-timer``


## 6. Use the MyTechRamblings.Templates within Visual Studio

> Be sure to have enabled the following option in Visual Studio: ``Tools > Options > Preview Features > Show all .NET Core templates in the New Project dialog``

Try to create a new project a the new templates should appear in ``Create new project`` dialog.

![list-templates](/img/dotnet-vs-list-templates.png)

If you select any of the template you have created a dialog will show up a dialog will pop-up and it will contain every parameter that you have specified in the ``ide.host.json``

- WebAPI Visual Studio dialog:

![api-vs-dialog](/img/dotnet-api-template-vs-dialog.png)

- Worker Service Visual Studio dialog:

![worker-vs-dialog](/img/dotnet-worker-template-vs-dialog.png)

- Azure function Visual Studio dialog:

![function-vs-dialog](/img/dotnet-function-template-vs-dialog.png)


# **Some useful links**

- https://github.com/dotnet/templating/wiki
- https://github.com/sayedihashimi/template-sample
- https://github.com/Dotnet-Boxed/Templates