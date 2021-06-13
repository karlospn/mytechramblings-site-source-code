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

But first let's talk about a few key concepts that are worth mentioning.

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
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action``, so it can be deployed either using ``Azure DevOps`` or ``GitHub``.

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
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action``, so it can be deployed either using ``Azure DevOps`` or ``GitHub``.

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
  - I have also included an ``Azure Pipelines`` YAML file and a ``GitHub Action``, so it can be deployed either using ``Azure DevOps`` or ``GitHub``.

In the following sections I'm going to convert these 3 apps in .NET templates, package them in a single NuGet package named "MyTechRamblings.Templates" and start using them.

## 2.1. Convert the NET 5 WebApi in a .NET template

###  **Create the template.json file**

The first step will be to create the ``template.json`` file, but before start building it you need to know what exactly you want to parameterize in the template.

After taking a look at the different feature I have built on the web api, I have come with a list of features that I want to parameterize:

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


As you see it's a pretty long list, but it tackles a lot of scenarios.
- If you are working with Docker just set the ``Docker`` parameter to true and a ready to go dockerfile will be created alongside with your api.
- If you want tests in you solution set the ``Tests`` parameter to true.
- If the api is going to be deployed with Azure Devops set the ``AzurePipelines`` attribute to true.
- If the api is going to be deployed with GitHub set the ``GitHub`` attribute to true.
- Depending of how the api is going to be deployed set the ``DeploymentType`` attribute and the pipelines will changed accordingly.
- Enable or disable features like ``HealthChecks``or ``Swagger``.
- If you are using Authorization with Azure Active Directory, set the ``Authorization`` attribute to true and set the ``AzureAdTenantId``, ``AzureAdDomain``, ``AzureAdClientId``, ``AzureAdSecret`` attributes accordingly.

There is a default value for each attribute. You can put a default value that matches with your most common scenario so you won't need to set all those values every time you want to create a new app.

## 2.2. Convert the NET 5 Worker Service in a .NET template

## 2.3. Convert the Azure Function in a .NET template

## 2.4. Create the MyTechRamblings.Templates NuGet package

## 2.5. Install and use the MyTechRamblings.Templates  with the .NET CLI

## 2.6. Use the MyTechRamblings.Templates within Visual Studio


# **Some useful links**

- https://github.com/dotnet/templating/wiki
- https://github.com/sayedihashimi/template-sample
- https://github.com/Dotnet-Boxed/Templates