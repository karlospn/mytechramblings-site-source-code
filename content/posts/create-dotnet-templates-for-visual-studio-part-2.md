---
title: "How to build a .NET template and use it within Visual Studio. Part 2: Creating a template package"
date: 2021-06-10T14:25:29+02:00
draft: true
tags: ["dotnet", "csharp", "templates", "vs", "visual", "studio"]
---

> This is a 2 part-series.
> - In **part 1** I talked about some key concepts that you should know when creating a .NET template.    
>   - If you want to read it, click [**here**](https://www.mytechramblings.com/posts/create-dotnet-templates-for-visual-studio-part-1/)
> - In **part 2** I will show you the process of converting a few .NET apps into .NET templates, package them together in a single NuGet file and use them as template within Visual Studio.


**Just show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/pack-dotnet-templates-example).   
Also I have upload the NuGet package to [nuget.org](https://www.nuget.org/packages/MyTechRamblings.Templates).


# **Creating the MyTechRamblings.Templates package**

In the following sections I will be converting 3 apps into templates, package them in a NuGet file named ``MyTechRamblings.Templates`` and showing you how to use them within Visual Studio.

The ``MyTechRamblings.Templates`` package will contain 2 solution templates and 1 project template. 

Before start coding remember what I said in part 1:

> - **Using custom .NET CLI templates within Visual Studio is only available in Visual Studio version 16.8 or higher**.    
> - **Also if you're creating a solution template you need at least Visual Studio version 16.10 or higher.**

# 1. Prerequisites:

**I have develop 3 apps beforehand**, that's because in this post I will focus on the process of converting these 3 apps in templates.

The 3 apps I have built are the following ones:
- **NET 5 Web Api**
  - It is an entire solution application. It uses a N-layer architecture with 3 layers:
    - ``WebApi`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The api has the following features already built-in:
    - ``HealthChecks``
    - ``Swagger``
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Identity.Web``
    - ``Dockerfile``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` YAML file to deploy the app into Azure App Service, thereby when you create a new solution using the template a ready-to-go deployment pipeline will be created.

  >If you want to take a look at the api source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/WebApiNet5Template)

- **NET 5 Worker Service that consumes RabbitMq messages**
  - It is an entire solution application. 
  - The application is a ``BackgroundService`` that consumes messages from a RabbitMq server. It uses a N-layer architecture with 3 layers:
    - ``Worker`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The service has the following features already built-in:
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Extensions.Hosting.Systemd``
    - ``Microsoft.Extensions.Hosting.WindowsServices``
    - ``Dockerfile``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` YAML file to deploy the app into Azure App Service, thereby when you create a new solution using the template a ready-to-go deployment pipeline will be created for you for you.

  >If you want to take a look at the worker source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/HostedServiceNet5RabbitConsumerTemplate)

- **NET Core 3.1 Azure Function that gets triggered by a timer**
  - This one is not an entire solution, instead it is just a single project. 
  - It is a NET Core 3.1 Azure Function that is triggered by a timer.
  - The function has the following features already built-in:
    - ``Dependency Injection``
    - ``Logging``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` YAML file to deploy the function into Azure Functions, thereby when you create a new solution using the template a ready-to-go deployment pipeline will be created for you.

  >If you want to take a look at the function source code, click [HERE](https://github.com/karlospn/pack-dotnet-templates-example/tree/main/src/AzureFunctionTimerProjectTemplate)


# 2. Convert the NET 5 Web Api into a template

## 2.1. **Create the template.json file**

The first step is to create the ``template.json`` file, but before start building it you need to know what you want to parameterize in the template.

After taking a look at the different features I have built on the api, I have come with a list of features that I want to parameterize.

| Parameter  Name           | Description                                                                                                                                                                                                                                                                                                | Default value           |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------|
| **Docker**                | Adds or removes a Dockerfile file.                                                                                                                                                                                                                                   | _true_                    |
| **ReadMe**                | Adds or removes a README markdown file describing the project.                                                                                                                                                                                                                                                     | _true_                    |
| **Tests**                 | Adds or removes an Integration Test project and a Unit Test project in the solution.                                                                                                                                                                                                                                                 | _true_                    |
| **GitHub**                | Adds or removes a GitHub Action file. This GitHub Action is used to deploy the api into an Azure Web App.                                                                                                                                                                                                                          | _false_                   |
| **AzurePipelines**        | Adds or removes an Azure pipeline YAML file. The pipeline is used to deploy the api into an Azure Web App.                                                                                                                                                                                                                   | _true_                    |
| **DeploymentType**        | Specifies how you want to deploy the api. The possible values are ``DeployAsZip`` or ``DeployAsContainer``.    Depending of the value you choose the content of the deployment pipeline will vary.    If you choose to not create neither a GitHub Action nor an Azure Pipeline this parameter is useless. | _DeployAsZip_             |
| **AcrName**               | An Azure ACR registry name. Only used if you are going to be deploying with container.                                                                                                                                                                                                                     | _acrcponsndev_            |
| **AzureSubscriptionName** | An Azure DevOps Service Endpoint subscription. Only used if deploy with Azure Pipelines.                                                                                                                                                                                            | _cponsn-dev-subscription_ |
| **AppServiceName**        | The name of the Azure App Service where the app will be deployed.                                                                                                                                                                                                                                         | _app-svc-demo-dev_        |
| **Authorization**         | Enables or disables the use of authorization using ``Microsoft.Identity.Web``                                                                                                                                                                                                                                              | _true_                    |
| **AzureAdTenantId**       | Azure Active Directory Tenant Id. Only necessary if ``Authorization`` is enabled.                                                                                                                                                                                                                              | _8a0671e2-3a30-4d30-9cb9-ad709b9c744a_                       |
| **AzureAdDomain**         | Azure Active Directory Domain Name. Only necessary if ``Authorization`` is enabled.                                                                                                                                                                                                                            | _cpnoutlook.onmicrosoft.com_                       |
| **AzureAdClientId**       | Azure Active Directory App Client Id. Only necessary if ``Authorization`` is enabled.                                                                                                                                                                                                                          | _fdada45d-8827-466f-82a5-179724a3c268_                       |
| **AzureAdSecret**         | Azure Active Directory App Secret Value. Only necessary if ``Authorization`` is enabled.                                                                                                                                                                                                                       | _1234_                       |
| **HealthCheck**           | Enables or disables the use of healthchecks.                                                                                                                                                                                                                                                                           | true                    |
| **HealthCheckPath**       | HealthCheck api path. Only necessary if ``HealthCheck`` is enabled.                                                                                                                                                                                                                                                | _/health_                 |
| **Swagger**               | Enables or disables the use of Swagger.                                                                                                                                                                                                                                                                                | _true_                    |
| **SwaggerPath**           | Swagger api path. Only necessary if ``Swagger`` is enabled.                                                                                                                                                                                                                                | _api-docs_                |
| **Contact**               | The contact details to use if someone wants to contact you. Only necessary if ``Swagger`` is enabled.                                                                                                                                                                                                          | _user@example.com_        |
| **CompanyName**           | The name of the company. Only necessary if ``Swagger`` is enabled.                                                                                                                                                                                                                                             | _mytechramblings_         |
| **CompanyWebsite**        | The website of the comany. Only necessary if ``Swagger`` is enabled.                                                                                                                                                                                                                                           | _www.mytechramblings.com_ |
| **ApiDescription**        | The description of the api. Only necessary if ``Swagger`` is enabled.                                                                                                                                                                                                                                          | _Put your api info here._  |


- If you are working with Docker just set the ``Docker`` parameter to ``true`` and a Dockerfile will be placed alongside your api.
- If you want to add some tests in your solution set the ``Tests`` parameter to true. A unit test project and a integration test project will be placed inside the ``/test`` folder.
- If the api is going to be deployed with Azure Devops set the ``AzurePipelines`` parameter to true. A YAML pipeline and a ``/pipelines`` folder will be added inside your solution. 
- If the api is going to be deployed with GitHub set the ``GitHub`` parameter to true. A YAML file containing a GitHub Action and a ``/.github`` folder will be added inside your solution.
- Depending of how the api is going to be deployed set the ``DeploymentType`` parameter accordingly. 
  - If you deploy using containers set the value to ``DeployAsContainer`` and the deployment pipeline will be updated into a container deployment pipeline.
  - If you deploy using a zip file set the value to ``DeployAsZip``, and the deployment pipeline will be updated into an artifact deployment pipeline.
- You can enable or disable features like ``HealthChecks``or ``Swagger``.
- If you are using Authorization with Azure Active Directory, set the ``Authorization`` parameter to true and set the ``AzureAdTenantId``, ``AzureAdDomain``, ``AzureAdClientId``, ``AzureAdSecret`` parameters accordingly. 

There is a default value for each parameter.    
You can use a default value that matches with your most used scenario, so you won't need to set all those parameters every time you want to scaffold a new app.

After listing which features I wanted to parameterize, here's the ``template.json`` file:

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

Let me make an in-depth rundown of the meaning of every value:

- ``author``: The author of the template.
- ``classifications``: A set of characteristics of the template that a user might search for it. 
  - The classification items will appear as tags in the .NET CLI. You can search templates by classification using the ``dotnet new --tag <CLASSIFICATION_ITEM>`` command.
  - You can search templates by classification withinn Visual Studio using the ``Project Type`` dropdown.
- ``name``: The name of the template. That's the name that will appear in the .NET CLI and in VS.
- ``description``:  The description of the template. The description will appear in the .NET CLI when you ran the ``dotnet new <TEMPLATE_NAME> -h`` command. In VS it will appear in the ``Create new project dialog``.
- ``groupIdentity``: The ID of the group this template belongs to.
- ``identity``: A unique ID for this template.
- ``shortName``: A short name used for selecting the template in the .NET CLI. The ``shortName`` appears when you run ``dotnet new -l`` and it is probably the one that you will use the most when you create a new solution using the template. To create a new solution with the api template you just need to execute the ``dotnet new mtr-api`` command.   
Right now it has no use in Visual Studio.
- ``defaultName``: The name that will be used when you create a new solution using the template if no name has been specified. 
- ``tags``: You can add multiple tags but at least you have to specify the ``language`` tag and the ``type`` tag.
  - The language tag specifies the programming language. 
  - The type tag specifies the type of the template project. The possible values are: ``project``, ``solution``, ``item``. 
- ``sourceName``: The template engine will look for any occurrence of the ``sourceName`` and replace it. It will rename files if there is an occurrence. It will also replace file content if there is an occurrence.   
I'm using  ``ApplicationName`` as the ``sourceName`` and naming every ``.csproj`` with the ``ApplicationName`` prefix:
  - ``ApplicationName.sln``
  - ``ApplicationName.WebApi.csproj``
  - ``ApplicationName.Library.Contracts.csproj``
  - ``ApplicationName.Library.Impl.csproj``
  - ``ApplicationName.Repository.Contracts.csproj``
  - ``ApplicationName.Repository.Impl.csproj``

  When you create a new solution using this template every ``.csproj`` file and the ``.sln`` file will be renamed by the template engine from ``ApplicationName`` to the name chosen by the user.   
  Also the namespace prefix inside all the csharp files will be renamed from ``ApplicationName`` to the name chosen by the user.

- ``preferNameDirectory``: Indicates whether to create a directory for the template. If you set the value to ``false`` the solution will be placed in your current directory.

- ``specialCustomOperations``: The templating engine supports conditional operators but it only supports them in a certain file types. If you need to add conditionals operators in another file types you need to add them here.    
In my template I want to add conditional operators on the YAML files, that  why I'm adding a custom operation that applies to all the yml files (``**/*.yml``)

- ``sources.modifiers`` : The sources.modifiers allows us to include or exclude files from the solution based on a condition.

- ``symbols``: The symbols section is where you specify the inputs you want to parameterize and also define the behaviour of those inputs. 

The ``symbols`` and the ``sources.modifiers`` section is the meat of the ``template.json`` file.

I'm not going to try to explain the meaning of every symbol that I have placed on the ``symbols`` section, mainly because it will be quite repetitive because most of the symbols are using exactly the same strategy.   
Instead of that, I'm going to explain the strategies I have used, so in the end every symbol from the ``symbols`` section will drop down in one of the strategies described in the next section. 

## **Symbol replacement**

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
When you try to create a new solution using this template:

- The template engine will try to find the ``HEALTHCHECK-PATH`` string anywhere in the template and if it finds it, it will be replaced either by the user input or the default value (``/health``).    
If you take a look at the ``Startup.cs``, you'll see the _``replaces``_ value is hard-coded in the template.

```csharp
   endpoints.MapHealthChecks("HEALTHCHECK-PATH", new HealthCheckOptions
    {
      ResponseWriter = ApplicationBuilderWriteResponseExtension.WriteResponse
    });
```

When the user creates a new solution, the template engine will find the magic string ``HEALTHCHECK-PATH`` and replaced it by the user input or the ``defaultValue``.

If you take a look at the symbols section from the  ``template.json`` file, you will find a lot of symbols using this very same strategy but with a different ``replaces`` value, this is because that's the easiest way to update certain parts of the source code with the value the user inputs.

## **Use conditional operators based on the symbol value**

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
- If the user sets the value to ``false``, the template engine needs to remove any ``Microsoft.Web.Identity`` reference from the solution.
- If the user sets this symbol to ``true``, the template engine needs to keep the references to the ``Microsoft.Web.Identity`` library.

If you take a look at the  ``Startup.cs``, you'll see that all the ``Microsoft.Web.Identity`` references are being wrapped in a conditional operator.

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

The ``Microsoft.Web.Identity`` references are only being kept if the symbol ``Authorization`` is set to ``true``.    
That's not enough, we need to remove the ``Microsoft.Web.Identity`` references from the ``appsettings.json`` file too.

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


## **Use a symbol value to compute another one**

- Given the value of a symbol, you can set the value of another one.

**Example:**

The ``DeploymentType`` symbol is of type ``choice`` and with its value you can set the value of the ``DeployContainer`` symbol and the ``DeployZip`` symbol.

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

The ``DeployContainer`` symbol and the ``DeployZip`` symbol are used to tailor the deployment pipeline. 
- If the user sets the ``DeploymentType`` symbol to ``DeployAsContainer``,  then the ``DeployContainer`` symbol is set to ``true``.    
The ``DeployContainer`` symbol value is used to create a deployment pipeline that will build and deploy a docker image.
- If the user sets the ``DeploymentType`` symbol to ``DeployAsZip``,  then the ``DeployZip`` symbol is set to ``true``.    
The ``DeployZip`` symbol value is used to create a deployment pipeline that will build and deploy a zipped artifact.

Take a look at how the Azure Pipelines YAML file uses the ``DeployAsZip`` and ``DeployAsContainer`` symbol value to add a specific pipeline type.

```yaml
trigger:
- master

pool: 
  vmImage: 'ubuntu-latest'

variables:
  - name: buildConfiguration
    value: 'Release'
  - name: azureSubscription
    value: 'AZURE-SUBSCRIPTION-ENDPOINT-NAME'
  - name: appServiceName
    value: 'APP-SERVICE-NAME'
#if (DeployContainer)
  - name: registryName
    value: 'ACR-REGISTRY-NAME'
#endif

#if (DeployContainer)
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
    packageForLinux: '$(build.artifactstagingdirectory)/*.zip'
#endif
```
Take a look at how the GitHub Action YAML file uses the ``DeployAsZip`` and ``DeployAsContainer`` symbol value to add a specific pipeline type.

```yaml
name: .NET api deploy to Azure App Service

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

env:
  AZURE_WEBAPP_NAME: APP-SERVICE-NAME   
#if (DeployContainer)
  CONTAINER_REGISTRY: ACR-REGISTRY-NAME.azurecr.io 
#endif

#if (DeployContainer)
jobs:
  build-and-deploy-to-dev:
    runs-on: ubuntu-latest
    environment: dev
    steps:
    # Checkout the repo
    - uses: actions/checkout@master

    # Authenticate to Azure
    - name: Azure authentication
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
 
    # Authenticate to ACR
    - name: ACR authentication
      uses: azure/docker-login@v1
      with:
        login-server: ${{ env.CONTAINER_REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }} 

    # Build and push the Docker image
    - name: Docker Build & Push to ACR
      run: |
        docker build . -t ${{ env.CONTAINER_REGISTRY }}/ApplicationName:${{ github.sha }}
        docker push ${{ env.CONTAINER_REGISTRY }}/ApplicationName:${{ github.sha }} 
    # Deploy to Azure
    - name: 'Deploy to Azure Web App for Container'
      uses: azure/webapps-deploy@v2
      with: 
        app-name: ${{ env.AZURE_WEBAPP_NAME }} 
        images: ${{ env.CONTAINER_REGISTRY }}/ApplicationName:${{ github.sha }}
#endif

#if (DeployZip)
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      # Checkout the repo
      - uses: actions/checkout@master
      
      # Setup .NET Core SDK
      - name: Setup .NET Core
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '5.0.x'
      
      # Run dotnet build and publish
      - name: dotnet build and publish
        run: |
          dotnet restore
          dotnet build --configuration Release
          dotnet publish -c Release -o './myapp' 
          
      # Deploy to Azure Web apps
      - name: 'Run Azure webapp deploy action using publish profile credentials'
        uses: azure/webapps-deploy@v2
        with: 
          app-name: ${{ env.AZURE_WEBAPP_NAME }} # Replace with your app name
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE  }} # Define secret variable in repository settings as per action documentation
          package: './myapp'
#endif
```

## **Excluding files based on a symbol value**

- The ``source.modifiers`` section can be combined with the ``symbols`` section to include or exclude existing files based on a symbol value.

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

- If the symbol value is set to ``false``, the template engine needs to remove the "Dockerfile" file from the solution.
- If the symbol value is set to ``true``, the template engine needs to keep the "Dockerfile" file.

To achieve the desired behaviour, you can add the following object inside the ``source.modifiers`` array.   
The ``Docker`` symbol value is used as the condition to exclude the file.

```javascript
  {
      "condition": "(!Docker)",
      "exclude":
      [
          "Dockerfile"
      ]
  }
```

The template engine will remove the "Dockerfile" file, if the symbol ``Docker`` equals to ``false``.


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

- If the symbol value is set to ``false``, the template engine needs to:
  - Remove the entire folder that contains the unit test project.
  - Remove the entire folder that contains the integration test project.
  - Remove the project references from the ``.sln`` file.
- If the symbol value is set to ``true``, the template engines needs to keep the test projects.

To achieve the desired behaviour, you can add the following object in the ``source.modifiers`` array.   

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
The ``Tests`` symbol value is used as the condition to exclude both test folders. To remove the test project references from the ``.sln`` file we can use a conditional operator.

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

## 2.2. **Create the dotnetcli.host.json file**

If your template needs some command line parameters you can customize them by adding a ``dotnetcli.host.json`` file inside the ``.template.config`` folder.

In the ``dotnetcli.host.json`` file you can specify the short and long names for each of the command line parameters.  

This step is not mandatory and in my case I have quite a sizeable list of parameters to customize, but nonetheless I prefer to customize each parameter name as I see fit.   
Here's how the ``dotnetcli.host.json`` looks:

```javascript
{
    "$schema": "http://json.schemastore.org/dotnetcli.host",
    "symbolInfo":
    {
        "Docker": {
            "longName": "dockerfile",
            "shortName": "d"
        },
        "ReadMe": {
            "longName": "readme",
            "shortName": "r"
        },
        "Tests": {
            "longName": "tests",
            "shortName": "t"
        },
        "GitHub": {
            "longName": "github",
            "shortName": "ga"
        },
        "AzurePipelines": {
            "longName": "azure-pipelines",
            "shortName": "ap"
        },
        "DeploymentType": {
            "longName": "deploy-type",
            "shortName": "dt"
        },
        "AcrName": {
            "longName": "acr-name",
            "shortName": "acr"
        },
        "AzureSubscriptionName": {
            "longName": "azure-subscription",
            "shortName": "az"
        },
        "AppServiceName": {
            "longName": "app-service-name",
            "shortName": "asn"
        },
        "Authorization": {
            "longName": "authorization",
            "shortName": "auth"
        },
        "AzureAdTenantId":{
            "longName": "aad-tenant-id",
            "shortName": "at"
        },
        "AzureAdDomain":{
            "longName": "aad-domain-name",
            "shortName": "ad"
        },
        "AzureAdClientId":{
            "longName": "aad-client-id",
            "shortName": "ac"
        },
        "AzureAdSecret":{
            "longName": "aad-secret-value",
            "shortName": "as"
        },
        "HealthCheck": {
            "longName": "healthcheck",
            "shortName": "hck"
        },
        "HealthCheckPath": {
            "longName": "healthcheck-path",
            "shortName": "hp"
        },
        "Swagger": {
            "longName": "swagger",
            "shortName": "s"
        },
        "SwaggerPath": {
            "longName": "swagger-path",
            "shortName": "sp"
        },
        "Contact": {
            "longName": "contact-mail",
            "shortName": "cm"
        },
        "CompanyName": {
            "longName": "company-name",
            "shortName": "cn"
        },
        "CompanyWebsite": {
            "longName": "company-website",
            "shortName": "cw"
        },
        "ApiDescription": {
            "longName": "api-description",
            "shortName": "desc"
        }
    }
}
```

Remember that there are default values defined in the ``template.json`` file for each parameter, so you don't have to specify each and every one of the command line parameters if you don't need to. 

If you want to create a solution using the default values an override only a couple of paramenters, you could totally do it.   
For example if I execute this command: ``dotnet new mtr-api --github true --azure-pipelines false``, it will create an app using the default values for every command line parameter except the ``github`` and ``azure-pipelines`` parameters.

## 2.3. **Create the ide.host.json file**

If your template needs some command line parameters and you want to use it within Visual Studio you need to add an ``ide.host.json`` file inside the ``.template.config`` folder.

This file will be used to show the command line parameters inside a project dialog when you try to create a new project.   

![api-vs-dialog](/img/dotnet-api-template-vs-dialog.png)

Also you can customize your templates appearance in the Visual Studio template list with an icon. If it is not provided, a default icon will be associated with your project template.

Here's how the ``ide.host.json`` for the api looks like:

```javascript
{
    "$schema": "http://json.schemastore.org/vs-2017.3.host",
    "order": 0,
    "icon": "icon.png",
    "symbolInfo": [
      {       
        "id": "Docker",
        "name": 
        {
          "text": "Adds a Dockerfile."
        },
        "isVisible": true      
      },
      {       
        "id": "ReadMe",
        "name": 
        {
          "text": "Adds a Readme."
        },
        "isVisible": true      
      },
      {
        "id": "Tests",
        "name": 
        {
          "text": "Adds a Unit Test and a Integration Test project."
        },
        "isVisible": true
      },
      {       
        "id": "GitHub",
        "name": 
        {
          "text": "Adds a GitHub action to deploy the application."
        },
        "isVisible": true      
      },
      {       
        "id": "AzurePipelines",
        "name": 
        {
          "text": "Adds an Azure Pipeline YAML to deploy the application."
        },
        "isVisible": true      
      },
      {       
        "id": "DeploymentType",
        "isVisible": true      
      },
      {       
        "id": "AcrName",
        "name": 
        {
          "text": "Name of the ACR Registry, only used if deploying with container."
        },
        "isVisible": true      
      },
      {       
        "id": "AzureSubscriptionName",
        "name": 
        {
          "text": "Azure subscription name."
        },
        "isVisible": true      
      },
      {       
        "id": "AppServiceName",
        "name": 
        {
          "text": "Azure Application Service Name."
        },
        "isVisible": true      
      },
      {       
        "id": "Authorization",
        "name": 
        {
          "text": "Enable the use of authorization with Microsoft.Identity.Web."
        },
        "isVisible": true      
      },
      {       
        "id": "AzureAdTenantId",
        "name": 
        {
          "text": "Azure Active Directory Tenant Id. Only necessary if Authorization is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "AzureAdDomain",
        "name": 
        {
          "text": "Azure Active Directory Domain Name. Only necessary if Authorization is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "AzureAdClientId",
        "name": 
        {
          "text": "Azure Active Directory App Client Id. Only necessary if Authorization is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "AzureAdSecret",
        "name": 
        {
          "text": "Azure Active Directory App Secret Value. Only necessary if Authorization is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "HealthCheck",
        "name": 
        {
          "text": "Enable the use of healthchecks."
        },
        "isVisible": true      
      },
      {       
        "id": "HealthCheckPath",
        "name": 
        {
          "text": "HealthCheck path. Only necessary if HealthCheck is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "Swagger",
        "name": 
        {
          "text": "Enable the use of Swagger."
        },
        "isVisible": true      
      },
      {       
        "id": "SwaggerPath",
        "name": 
        {
          "text": "Swagger UI Path. Only necessary if Swagger is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "Contact",
        "name": 
        {
          "text": "The contact details to use if someone wants to contact you. Only necessary if Swagger is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "CompanyName",
        "name": 
        {
          "text": "The name of the company. Only necessary if Swagger is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "CompanyWebsite",
        "name": 
        {
          "text": "The website of the company. Only necessary if Swagger is enabled."
        },
        "isVisible": true      
      },
      {       
        "id": "ApiDescription",
        "name": 
        {
          "text": "The description of the WebAPI. Only necessary if Swagger is enabled."
        },
        "isVisible": true      
      }
    ]
}
```

# 3. Convert the remaining 2 apps into templates

- We still need to convert the ``Worker Service`` application and the ``Azure Function`` into a .NET template.

The conversion process is exactly the same as the one described for the webapi, so I'm not going to bother writing about it or this post is going to drag on forever.   

In both case the ``template.json`` file is quite similar, if you're interested in the end result:
- Here`s the [link](https://github.com/karlospn/pack-dotnet-templates-example/blob/main/src/HostedServiceNet5RabbitConsumerTemplate/.template.config/template.json) to the ``template.json`` file for the ``Worker Service`` application.
- Here's the [link](https://github.com/karlospn/pack-dotnet-templates-example/blob/main/src/AzureFunctionTimerProjectTemplate/.template.config/template.json) to the ``template.json`` file for the ``Azure Function``.

# 4. Create the MyTechRamblings.Templates NuGet package

In **Part 1** I told you that there are a couple of ways for creating a template pack:
- Using a ``.csproj`` file and the ``dotnet pack`` command.
- Using a ``.nuspec`` file and the ``nuget pack`` command from ``nuget.exe``.

I'm using a ``.nuspec`` file. It looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package>
  <metadata>
    <id>MyTechRamblings.Templates</id>
    <version>0.2.0</version>
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
I don't want to create the template package manually, so I have created a Powershell script that fetches the nuget executable from the ``nuget.org`` website, bundles everything together and creates the package using the ``nuget pack`` command.

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

After running the script you will find the .nupkg inside  the ``/template/nuget`` folder.

# 5. Install and use the MyTechRamblings.Templates with the .NET CLI

To Install the package you can run this command:

-  ``dotnet new -i MyTechRamblings.Templates::0.2.0``   

The command will try to fetch the NuGet from ``nuget.org`` and install it. 

Or if you have the .nupkg in your local filesystem, you can ran this command:

-  ``dotnet new -i .\MyTechRamblings.Templates.0.2.0.nupkg``

After installing the templates pack you should run the ``dotnet new -l`` command and verify that the 3 templates appear on the list (``mtr-api``, ``mtr-rabbit-worker``, ``mtr-az-func-timer``).

![dotnet-list-templates](/img/dotnet-list-installed-templates.png)

Now you can create a new solution using the api template running the command:

-  ``dotnet new mtr-api``

This command will create a solution using the default values. If you want to specify a concrete parameter execute the ``dotnet new mtr-api -h`` command to list every command line  parameter available.

Or you can create a new solution using the worker service template running the command:

- ``dotnet new mtr-rabbit-worker``

This command will create a solution using the default values. If you want to specify a concrete parameter execute the ``dotnet new mtr-rabbit-worker -h`` command to list every command line parameter available.

Or you can create an Azure Function project using the azure function template running the command: 

- ``dotnet new mtr-az-func-timer``

This command will create a project using the default values. If you want to specify a concrete parameter execute the ``dotnet new mtr-az-func-timer -h`` command to list every command line  parameter available.

# 6. Use the MyTechRamblings.Templates within Visual Studio

> Be sure to enable the following option in Visual Studio: ``Tools > Options > Preview Features > Show all .NET Core templates in the New Project dialog``.   
> And remember:
> - **The MyTechRamblings.Templates package uses a couple of solution templates, so if you want to use it you need at least Visual Studio version 16.10 or higher.**

Try to create a new project withing Visual Studio and the new templates should appear in the ``Create new project`` dialog.   
If the templates doesn't show up, just search for them in the search box.

![list-templates](/img/dotnet-vs-list-templates.png)

If you select any of the templates a dialog will show up and it will contain every parameter that you have specified in the ``ide.host.json``.

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