---
title: "How to pack multiple .NET Core templates in a single NuGet package and use it within Visual Studio or the .NET CLI"
date: 2021-06-10T14:25:29+02:00
draft: true
tags: ["dotnet", "csharp", "templates", "vs", "visual", "studio"]
---

> **Show me the code**   
As always if you don’t care about the post I have upload the source code on my [Github](https://github.com/karlospn/pack-dotnet-templates-example).   
Also I have upload the NuGet package to [nuget.org](https://www.nuget.org/packages/MyTechRamblings.Templates).

When you install the .NET SDK you receive over a dozen built-in templates for creating projects  like console apps, web apis, class libraries, unit test projects, etc.

You can also create custom templates and install them using a NuGet package. In fact packaging .NET templates inside a NuGet it's nothing new, it's been around for quite a long time and it's a nice way to quickly scaffold your own applications.   

Everytime you need to create a new application you have to setup each and everytime a set of common features like: logging, swagger, healthchecks, authentication, create the dockerfile, create CI/CD pipelines, etc.  
That is in fact very time consuming and super boring, so a better approach is to create a template a reuse it everytime time that you need to create that type of application.

Anyways as I stated before, creating .NET templates and using NuGet to package them it's nothing new. _But then, why writing this post right now?_

Just hear me out...

A few years ago the experience to create and package reusable .NET templates was pretty bare bones, but little by little the experience has been enhanced.   

In fact a couple of weeks ago Microsoft anounced the release of Visual Studio 2019 version 16.10. This version add the capabilit of creating entire solutions from a .NET template, so even nowadays Microsoft keeps improving the templating experience.

Right now if you want to create a .NET template, the template engine offers features like:
- Replace values.
- Include and exclude files.
- Preprocessor directive to include or exclude entire blocks of code.
- Execute custom processing operations when your template is used.

When you want to use your own .NET template you can use it within Visual Studio and it doesn't matter if it's a project or a solution template. And also it's also fully integrated with the .NET CLI.

I think right now the experience when you want to create a template is quite good so it is a good moment to write about it.   

### **In this post I want to focus on the process of converting a few application in templates, package them in a NuGet and re-use it within Visual Studio.**   

But first let's talk about a few key concepts that you need to know.

# **1. Key Concepts**

## 1.1. **.template.config folder**

Templates are recognized by a special folder that exist at the root of your template. This  folder needs to be named specifically: _".template.config."_

When you create a template, all files and folders in the template folder are included as part of the template except for the .template.config folder. 

You create a NuGet with multiple templates, but each template needs to have a .template.config folder at root level.

Here's an example:

```bash
---src
    +---AzureFunctionTimerProjectTemplate
    |   +---.template.config
    |   +---src
    |   \---test
    +---HostedServiceNet5RabbitConsumerTemplate
    |   +---.template.config
    |   +---src
    |   \---test
    \---WebApiNet5Template
        +---.template.config
        +---src
        \---test
```

In this example the ``/src`` folder contains 3 templates and each template contains a .template.config folder.

Inside the .template.config folder you need **AT LEAST** a file named "template.json"

### **template.json**

The template.json file provides configuration information to the template engine. The minimum configuration requires the members shown in the following table, which is sufficient to create a functional template.

Here's a brief example about how a ``template.json`` looks like:

```javascript
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Travis Chau",
  "classifications": [ "Common", "Console" ],
  "identity": "AdatumCorporation.ConsoleTemplate.CSharp",
  "name": "Adatum Corporation Console Application",
  "shortName": "adatumconsole"
}
```

I will walk you through a few template.json files on **section 2**.

> If you want to know more about it, this page describes in-depth the attributes and the options that are available when creating a template.json file.   
https://github.com/dotnet/templating/wiki/Reference-for-template.json

## **1.2. Packaging your templates into a nuget package**

A template pack, in the form of a .nupkg NuGet package, requires that **all templates be stored in the content folder** within the package. 

There are a couple of ways of doing the packing:
- With the ``dotnet pack`` command and a .csproj file.
- With ``nuget pack`` command of Nuget.exe and a .nuspec file.

If you use the ``.csproj file`` approach you need to know that the .csproj is slightly different from a traditional code-project .csproj file. Here's an example:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <PackageType>Template</PackageType>
    <PackageVersion>1.0.0</PackageVersion>
    <PackageId>Someone.Templates</PackageId>
    <Title>The Templates</Title>
    <Authors>Me</Authors>
    <Description>Templates to use when creating an application</Description>
    <PackageTags>dotnet;templates;foo</PackageTags>
    <TargetFramework>netstandard2.0</TargetFramework>

    <IncludeContentInPack>true</IncludeContentInPack>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <ContentTargetFolders>content</ContentTargetFolders>
  </PropertyGroup>

  <ItemGroup>
    <Content Include="src\**\*" Exclude="src\**\bin\**;src\**\obj\**" />
    <Compile Remove="**\*" />
  </ItemGroup>
</Project>
```
Note the following settings:
- The ``<PackageType>`` setting is added and set to Template.
- The ``<PackageVersion>`` setting is added and set to a valid NuGet version number.
- The ``<PackageId>`` setting is added and set to a unique identifier. This identifier is used to uninstall the template pack and is used by NuGet feeds to register your template pack.
- The ``<TargetFramework>`` setting must be set, even though the binary produced by the template process isn't used. In the example below it's set to netstandard2.0.
- The ``<IncludeContentInPack>`` setting is set to true to include any file the project sets as content in the NuGet package.
The ``<IncludeBuildOutput>`` setting is set to false to exclude all binaries generated by the compiler from the NuGet package.
The ``<ContentTargetFolders>`` setting is set to content. This makes sure that the files set as content are stored in the content folder in the NuGet package. This folder in the NuGet package is parsed by the dotnet template system.
  
If you use the ``.nuspec file`` approach, the current nuspec.xsd schema file can be found [here](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.Packaging/compiler/resources/nuspec.xsd)   
Here's an example:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package>
  <metadata>
    <id>Someone.Templates</id>
    <version>1.0.0</version>
    <description>Templates to use when creating an application</description>
    <authors>Me</authors>
    <title>The Templates</title>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <license type="expression">MIT</license>
    <tags>dotnet templates</tags>
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

## **1.3. Installing the templates package**

Template packages are represented by a NuGet package (.nupkg) file. 
Like any NuGet package, you can upload the template pack to a NuGet feed. The dotnet new -i command supports installing template pack from a NuGet package feed or you can install a template pack from a .nupkg file directly.

- To install a template from a NuGet package stored at nuget.org:
```bash
dotnet new -i <NUGET_PACKAGE_ID>
```

- To install a template from a file system directory:
```bash
dotnet new -i <FILE_SYSTEM_DIRECTORY>
```

- To uninstall a template:
```bash
dotnet new -u <NUGET_PACKAGE_ID>
```

- To check if you template has been installed successfully:
```bash
dotnet new -l
```
The ``dotnet new -l`` command will list all the templates you have installed in your machine. If your custom template has beed installed successfully it will appear on the list.

![dotnet-list-templates](/img/dotnet-list-installed-templates.png)

## **1.4. Using your custom templates with the .NET CLI**

After you have installed your template you are good to go, you can create a project using your custom template running the ``dotnet new <TEMPLATE_NAME>`` command.

If your template needs some command options you can rely on the template engine to pick the values for you or you can customize it.   
If you want to customize the options you can add a ``dotnetcli.host.json`` file inside the ``.template.config`` folder.   
It's a pretty simple file were you can specify the custom short and long names for each of the options.

Here's an example:
```javascript
{
    "$schema": "http://json.schemastore.org/dotnetcli.host",
    "symbolInfo": {
        "FunctionName": {
            "longName": "function-name",
            "shortName": "fn"
        },
        "LogLevel": {
            "longName": "log-level",
            "shortName": "log"
        },
        "AcrName": {
            "longName": "acr-name",
            "shortName": "an"
        }
    }
}
```

With this ``dotnetcli.host.json`` file in place I can create my template running the following command:

```bash
dotnet new <TEMPLATE_NAME> -fn "MyFunc" -log "Debug" -an "acrappdev"
dotnet new <TEMPLATE_NAME> --function-name "MyFunc" --log-level "Debug" --acr-name "acrappdev"
```

If you don't create a ``dotnetcli.host.json`` the template engine will pick the command options for you, so you might like it or not.


## **1.6. Configure your template to work with Visual Studio**

If your template needs some command options you need to create a file named ``ide.host.json`` and place it inside the ``.template.config`` folder.   
This file will allow us to set up all of our command options to show up in the Visual Studio project dialog.

Here's an example:
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
      }
    ]
}
```
- The ``order`` attribute is used to specify the order of the template as shown in the New Project dialog.
- You can customize your template's appearance in the template list with an icon.  If it is not provided, a default generic icon will be associated with your project template. To add your own icon you just need to put an   ``icon.png`` file inside the ``.template.confg`` folder.
- The ``symbolInfo`` array contains all the command options we want to show in the Visual Studio dialog.
- The ``symbolInfo.isVisible`` property enables the parameter visibility. The default value of the ``isVisible`` property is ``false``.You need to change it to ``true`` or is not going to show in the Visual Studio dialog.


## **1.6. Using your templates within in Visual Studio**

> - **This feature is only available in Visual Studio version 16.8 or higher**.    
> - **If you're creating a solution template you need at least Visual Studio version 16.10 or higher.**

Right now there is **no way to install a template package within Visual Studio**, to install the templates on your machine **you need to do it using the .NET CLI**. _More info about how you can do it on section 1.3._

After you have installed the templates via .NET CLI you need to enable the use of templates within Visual Studio.

To enable this option visit the "Preview Features" options in the ``Tools > Options`` menu and look for the ``Show all .NET Core templates in the New Project dialog`` checkbox. 

After enabling this option and choosing to create a new project you’ll see your custom templates in the list of  available templates.
![list-templates](/img/dotnet-vs-list-templates.png)   

If you try to create a new project using your custom template a nice dialog will show up with all the options you have set in the ``ide.host.json`` file.    
Here's an example about how it looks:   
![api-vs-dialog](/img/dotnet-api-template-vs-dialog.png)

I think that's more than enough theory for now. Let's start building our package tenplates.

# **2. Creating the MyTechRamblings.Templates package**

I'll be creating some solution templates a some project templates and use it within Visual Studio, so remember what I said before:

> - **Using .NET CLI templates in Visual Studio is only available in Visual Studio version 16.8 or higher**.    
> - **Also if you're creating a solution template you need at least Visual Studio version 16.10 or higher.**

## Prerequisites:

**I have develop 3 apps beforehand**, that's because in this post I want to focus on the process of converting those 3 apps in templates, package them in a NuGet and use it within Visual Studio.

Those 3 apps are the following ones:
- **NET 5 Web Api**
  - It is an entire solution application. It uses a N-layer architecture with 3 layers:
    - ``WebApi`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The api also has the following features already built-in:
    -  ``Azure Pipelines`` YAML file.
    - ``GitHub Action`` file.
    - ``HealthChecks``
    - ``Swagger``
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Identity.Web``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` that allow us to deploy the api into an Azure App Service.

- **NET 5 Worker Service that consumes RabbitMq messages**
  - It is an entire solution application. It creates a ``BackgroundService`` that consumes messages from a RabbitMq server. It uses a N-layer architecture with 3 layers:
    - ``WebApi`` layer.
    - ``Library`` layer.
    - ``Repository`` layer.
  - It uses the ``Microsoft.Build.CentralPackageVersions`` MSBuild SDK. This SDK allows us to manage all the NuGet package versions in a single file. The file can be found in the ``/build`` folder.
  - The service also has the following features already built-in:
    - ``Serilog``
    - ``AutoMapper``
    - ``Microsoft.Extensions.Hosting.Systemd``
    - ``Microsoft.Extensions.Hosting.WindowsServices``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action``that allow us to deploy the service into an Azure App Service.

- **NET Core 3.1 Azure Function that gets triggered by a timer**
  - It is a single project. It creates a NET Core 3.1 Azure Function that is triggered by a timer.
  - The apps also has the following features already built-in:
    - HealthChecks
    - Swagger
    - Serilog
    - AutoMapper
    - Microsoft.Identity.Web
  - The function also has the following features already built-in:
    - ``Dependency Injection``
    - ``Logging``
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action`` that allow us to deploy the function.

In the following sections I'm going to convert these 3 apps in .NET templates, package them in a single NuGet package named "MyTechRamblings.Templates" and start using them.

## 2.1. Convert the NET 5 Web API in a template

###  **Create the template.json file**

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

- ``sources.modifiers`` : The sources.modifiers allows us to include or exclude files based on a condition.

- ``symbols``: The symbols section is where you specify the template inputs and define what those inputs should do. 

In our case the ``symbols`` section and the ``source.modifiers`` section are intertwined.
- You define a symbol in the ``symbol section`` and  the value from the symbol is used as a condition in the ``source.modifier`` section.

This is the meat of the ``template.json`` file so let's take a more in-depth look.

### **Symbol replacement**

- It replaces a value with the symbol value.

Example: 

I have the symbol "HealthCheckPath" of type "parameter" with a default value of "/health" and a replace value of "HEALTHCHECK-VALUE". 
```javascript
  "HealthCheckPath": {
    "type": "parameter",
    "datatype": "string",
    "defaultValue": "/health",
    "replaces": "HEALTHCHECK-PATH",
    "description": "HealthCheck path. Only necessary if HealthCheck is enabled."
  }
```
This symbol will try to find the "HEALTHCHECK-PATH" string anywhere in the template, and if they find it, it will be replaced either by the user input or the default value (/health) 
If you take a look in the ``Startup.cs``, you'll see the "replace" value.

```csharp
   endpoints.MapHealthChecks("HEALTHCHECK-PATH", new HealthCheckOptions
    {
      ResponseWriter = ApplicationBuilderWriteResponseExtension.WriteResponse
    });
```


### **Removing entire blocks of code based on the symbol value**

- This feature allows us to skip entire chunks of code based on a symbol value.

Example:
I have the symbol "Authorization" of type "bool" with the default value of "true".

```javascript
  "Authorization": {
    "type": "parameter",
    "datatype": "bool",
    "defaultValue": "true",
    "description": "Enables the use of authorization with Microsoft.Identity.Web."
  },
```

- If the user sets the symbol to "false" we need to remove any "Microsoft.Web.Identity" reference from the source code.
- If the user sets this symbol to "true" we keep the references to the "Microsoft.Web.Identity" Nuget.

If you take a look again at the  ``Startup.cs``:

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

As you can see we are keeping the "Microsoft.Web.Identity" code only if the symbol is set to true. Also we need to remove the references from the ``appsettings.json`` file.

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

And from the controller itself:

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

## Use a symbol value to compute another symbol value

- Given a specific value of a symbol you can set another symbol value.

Example:

The "DeploymentType" symbol is of type "choice" and with the symbol value you can setup the "DeployContainer" symbol and the "DeployZip" symbol.

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

Those 2 symbols can be used to tailor the deployment pipeline. If you take a look at the Azure DevOps pipeline YAML you'll see the following steps:
- If the symbol "DeploymentContainer" is set to true, use a container image build pipeline.
- If the symbol "DeployZip" is set to true, use an artifact build pipeline.

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

- You can combine the ``source.modifiers`` section with the ``symbols`` section to remove files from the template based on a symbol value.

Example 1:

The "Dockerfile" symbol is of type "parameter" and the default value is "true"

```javascript
  "Docker": {
      "type": "parameter",
      "datatype": "bool",
      "description": "Adds an optimised Dockerfile to add the ability to build a Docker image.",
      "defaultValue": "true"
  },

```

- If the symbol value is set to "false", we need to remove the "Dockerfile" file from the template.
- If the symbol value is set to "true", we need to keep the "Dockerfile" file.

We can achieve it adding the following object in ``source.modifiers`` array.

```javascript
  {
      "condition": "(!Docker)",
      "exclude":
      [
          "Dockerfile"
      ]
  }
```

Example 2:

The "Test" symbol is of type "parameter" and the default value is "true"

```javascript
  "Tests": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "true",
      "description": "Adds an integration and unit test projects."
  }
```

- If the symbol value is set to "false", we need to remove the entire Unit Test and Integration Test from the solution.
- If the symbol value is set to "true", we need to keep the Test projects.

We can achieve it adding the following object in ``source.modifiers`` array.

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
But also we need to exclude the test project from the ``.sln`` file, we can achieve it using a conditional check

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


## 2.2. Convert the NET 5 Worker Service in a .NET template

## 2.3. Convert the Azure Function in a .NET template

## 2.4. Create the MyTechRamblings.Templates NuGet package

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


## 2.5. Install and use the MyTechRamblings.Templates with the .NET CLI


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


## 2.6. Use the MyTechRamblings.Templates within Visual Studio

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