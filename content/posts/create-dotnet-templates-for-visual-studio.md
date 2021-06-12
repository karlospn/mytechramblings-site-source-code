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

I think right now the experience when you want to create a template is quite good so it is a good moment to write about it. **In this post I will focus on how you can create and pack multiple .NET Core templates in a single NuGet package and how to use it within Visual Studio or the .NET CLI.**

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

Also you can add an icon that will be displayed when you create a new project using your template, To achieve it you just need to put and ``icon.png`` file inside the ``.template.confg`` folder.

<ADD-IMG>


## **1.6. Using your templates within in Visual Studio**

> - **This feature is only available in Visual Studio 16.8 or higher**.    
> - **If you're creating a solution template you need at least VS 16.10 or higher.**

Right now there is no way to install a template package within Visual Studio, to install the templates on your machine you need to do it using the .NET CLI. _More info about how you can do it on section 1.3._

After you have installed the templates via .NET CLI you need to enable the use of templates within Visual Studio.

To enable this option visit the "Preview Features" options in the ``Tools > Options`` menu and look for the ``Show all .NET Core templates in the New Project dialog`` checkbox.

After enabling this option and choosing to create a new project you’ll see your custom templates in the list of  available templates.

---

I think that's enough concepts for now. Let's start building something.

# **2. Creating the MyTechRamblings.Templates package**

VS 16.9 didn't properly support full solution templates and only supported project templates.    

It's the newest release of Visual Studio (VS 16.10) that adds a user interface for creating solutions from dotnet  templates

# **Some useful links**

- https://github.com/dotnet/templating/wiki
- https://github.com/sayedihashimi/template-sample
- https://github.com/Dotnet-Boxed/Templates