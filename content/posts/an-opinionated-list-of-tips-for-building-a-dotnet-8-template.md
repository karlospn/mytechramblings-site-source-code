---
title: "My opinionated list of tips for building a .NET 8 app template"
date: 2023-12-25T14:36:32+01:00
description: "TBD"
tags: ["dotnet", "csharp", "templates"]
draft: true
---

When you install the .NET SDK, it comes equipped with a variety of templates that are readily available for your use. However, these templates are essentially bare, serving as a foundational starting point for your projects.

[.NET Aspire](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview) embodies a similar philosophy. It is an opinionated, cloud ready stack for building applications, offering a preconfigured set of tools such as HealthChecks and OpenTelemetry. You can take a look at the Aspire templates in [here](https://github.com/dotnet/aspire/tree/main/src/Aspire.ProjectTemplates)

In the realm of enterprise development, I usually tend to find that there aree a set of opinionated rules defined when you have to build an application. These rules might dictate the use of specific application architectures (e.g., Clean Architecture or Vertical Slice) or mandate the utilization of private NuGet packages for functionalities like logging or configuration. This is where .NET Aspire has limitations, as, despite its well-implemented nature, it remains a generic template. Consequently, regardless of how robust .NET Aspire may be, it doesn't cater to the specific, tailored needs of individual enterprises, this is where creating and use your own templates that enforce your organization's rules becomes a valuable approach when starting the development of a new application.

I have been building .NET templates for quite some time and for multiple companies. As a result, I thought it might be helpful to compile a list of tips to consider when creating a .NET app template. 

Keep in mind that this is an **opinionated list**, reflecting my own ideas. You may or may not agree with some of the points. Undoubtedly, what I consider important might not hold the same significance for you, and vice versa. 

Furthermore, I'm sure I'll overlook a few points that I consider crucial, but at the time of creating this list, they may have slipped my mind. 

So, without further ado, let's get started.


# **Don't try to build a single template with everything**

The first tip is very simple. Don't try to create a single template that does everything, like encompassing multiples architectures (e.g. Clean Architecture or N-layer architecture) or multiple kind of apps (e.g. APIs, Worker Servies).

Instead of that, it is much better to have several templates where each one fulfills a concrete objective. That way it is much more easy to maintain an update them.

When it comes to distributing your templates, it doesn't matter if you have 1 or 10; the distribution process will be exactly the same. More info, in the next tip.


# **Pack all your .NET templates into a single NuGet package**

The best way to distribute your custom .NET app templates is via NuGet package. You can package has many templates as you want in a single package, but each template needs to have a ``.template.config`` folder at its root level. 

Inside the ``.template.config`` folder you need to place at least a file named ``template.json``. This file provides information about what the template engine should do with your template

Within the NuGet package all the templates must be stored in the ``/content`` folder.

Here’s how the folder structure should look like:

```shell
---content
    +---TemplateA
    |   +---.template.config
    |   +---src
    |   \---test
    +---TemplateB
    |   +---.template.config
    |   +---src
    |   \---test
    \---TemplateC
        +---.template.config
        +---src
        \---test
```


Also, when you install or uninstall a template package, all templates contained in the package will be added or removed, respectively.

Once you have created your template package, you can upload it to your company's feed (e.g., Azure Artifacts, GitHub Packages, Artifactory), allowing everyone to start using it right away.


# **Add the template package version as a tag**

When creating your template package, include the package version as a tag in the ``.template.config`` file. This becomes especially useful when you can't recall the specific version of the package template installed on your machine.

This step can be automated in a dozen different ways, making it a straightforward process.

The ``.template.config`` file should end up looking like this:
```json
{
    "$schema": "http://json.schemastore.org/template",
    "author": "mytechramblings.com",
    "classifications": [
      "Cloud",
      "Web",
      "WebAPI",
      "v1.3.0"
    ],
    "name": "My TechRamblings NET 8 WebApi",
    "description": "A WebApi solution.",
    ...
}
```

Now, by listing the available templates using the ``dotnet new list`` command, I can easily identify the versions of my custom .NET templates installed on my computer.

<add-img>


# **Centralize the NuGet package versions using a Directory.Packages.props files**

This tip applies if you're building a template that is comprised of multiple projects (e.g., a Clean Architecture template).

Managing dependencies for multi-project templates in the long run, as new package versions become available, can be challenging. It often leads to spending a significant amount of time dealing with projects using different versions of the same package within a single application.

Using this feature in your .NET template, especially if you're building a multi-project template, is a good idea. Also, it is really easy to implement.

Create a ``Directory.Packages.props`` file in the root of your template, similar to the following example:

```xml
<Project>
  <ItemGroup Label="Microsoft Nugets">
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="8.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="8.0.0" />
    <PackageVersion Include="Microsoft.Identity.Web" Version="2.16.0" />
    <PackageVersion Include="Asp.Versioning.Mvc " Version="8.0.0" />
    <PackageVersion Include="Asp.Versioning.Mvc.ApiExplorer" Version="8.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
    <PackageVersion Include="System.Text.Json" Version="8.0.0" />
    <PackageVersion Include="System.ComponentModel.Annotations" Version="5.0.0" />
  </ItemGroup>

  <ItemGroup Label="Third party Nugets">
    <PackageVersion Include="AutoMapper" Version="12.0.1" />
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    <PackageVersion Include="Swashbuckle.AspNetCore.Annotations" Version="6.5.0" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageVersion Include="Hellang.Middleware.ProblemDetails" Version="6.5.1" />
  </ItemGroup>

  <ItemGroup Label="Test Nugets">
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.8.0"/>
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
    <PackageVersion Include="NSubstitute" Version="5.1.0" />
    <PackageVersion Include="xunit" Version="2.6.3"/>
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.5.5">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
     </PackageVersion >
    <PackageVersion Include="coverlet.collector" Version="6.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageVersion >
  </ItemGroup>
</Project>
```

In every project, set the property ``ManagePackageVersionsCentrally`` to ``true`` and then define which packages from the ``Directory.Packages.props`` are you going to use.

In each project, set the property ``ManagePackageVersionsCentrally`` to ``true`` and then define which packages from the ``Directory.Packages.props`` file you want to use. Here's an example:


```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

	<ItemGroup>
        <PackageReference Include="Asp.Versioning.Mvc.ApiExplorer" />
        <PackageReference Include="Asp.Versioning.Mvc " />
        <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" />
        <PackageReference Include="Microsoft.Identity.Web" />
        <PackageReference Include="Swashbuckle.AspNetCore" />
        <PackageReference Include="Swashbuckle.AspNetCore.Annotations" />
        <PackageReference Include="Serilog.Sinks.Console" />
        <PackageReference Include="Hellang.Middleware.ProblemDetails" />
	</ItemGroup>

</Project>
````

# **Use a Directory.build.props to manage the common properties required in all your projects**

This tip, like the last one, applies if you're building a template that is comprised of multiple projects (e.g., a Clean Architecture template).

If you take a look at the various projects constituting your .NET template, you will see a set of properties repeated in each of them. For instance, the ``LangVersion`` property, the ``Nullable`` property, or the ``ImplicitUsings`` property.

It is much better to create a ``Directory.build.props`` file at the root level of your template to manage the common  settings required in all your projects.

The next code snippet, shows the ``Directory.build.props`` file I am currently using for my templates:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>True</TreatWarningsAsErrors>
    <IsPackable>false</IsPackable>
    <NuGetAudit>true</NuGetAudit>
    <NuGetAuditMode>direct</NuGetAuditMode>
    <NuGetAuditLevel>high</NuGetAuditLevel>
  </PropertyGroup>
</Project>
```

# **Add Visual Studio support**

Adding support for Visual Studio to your .NET templates can be a very useful feature for developers utilizing this IDE, and it is remarkably simple to implement. Create a file named ``ide.host.json`` and place it inside the ``.template.config`` folder.   
This file will be utilized to display command-line parameters within a project dialog in Visual Studio when attempting to create a new project.

To employ it, you require at least Visual Studio version 16.10 or higher.

# **Every project that constitutes your app template should start with the exact same name**

If you're creating a multi-project .NET template, each project should begin with the exact same name.

Why? Let's consider building a Clean Architecture API .NET template; the folder structure will look (more or less) something like this:

```shell
MyTechRamblings.slm
---src
    +---MyTechRamblings.WebAPI
    |   +---MyTechRamblings.WebApi.csproj
    |   +---Program.cs
    |   \---appsettings.json
    +---MyTechRamblings.Application
    |   +---MyTechRamblings.Application.csproj
    +---MyTechRamblings.Core
    |   +---MyTechRamblings.Core.csproj
    |   +--Interfaces
    |   +--Services
    |   \--Aggregates    
    \---MyTechRamblings.Infrastructure
        +---MyTechRamblings.Infrastructure.csproj
        \---Data
            +--Queries
            \--AppDbContext.cs
```

As you can see, the solution (sln), every folder name, and every .csproj file that I have created begins with the "MyTechRamblings" prefix.   
This consistency is because I will be utilizing the ``sourceName`` property from the ``template.json`` file to rename the prefix based on whatever value the user provides when creating a new app.

The ``sourceName`` property value is replaced with the value provided using the ``-n``, ``--name`` options when running a template. The template engine will look for any occurrence of the name present in the config file and replace it in file names and file contents. If no name is specified by the host, the current directory is used.

Now, imagine that I have finished building my Clean Architecture .NET template and opted to package it into a NuGet package. The ``template.json`` file for this template includes the following properties:

```json
{
    ...
    "shortName": "mtr-api",
    "sourceName": "MyTechRamblings"
    ...
}
```

When creating a new app using the template, I will use the command: ``dotnet new mtr-api --name Foo``. The ``--name`` value will replace the "MyTechRamblings" string throughout the template, resulting in the following application:

```shell
Foo.slm
---src
    +---Foo.WebAPI
    |   +---Foo.WebApi.csproj
    |   +---Program.cs
    |   \---appsettings.json
    +---Foo.Application
    |   +---Foo.Application.csproj
    +---Foo.Core
    |   +---Foo.Core.csproj
    |   +--Interfaces
    |   +--Services
    |   \--Aggregates    
    \---Foo.Infrastructure
        +---Foo.Infrastructure.csproj
        \---Data
            +--Queries
            \--AppDbContext.cs
```

As you can see, the new application wil lbe correctly renamed based on the specified input.

# **Make use of the template built-in conditional pragmas to or disable certain features**

Not every application needs to have the same features, and you can control this variability using the built-in conditional pragmas.

Authorization is one of them. Depending on the type of API you're building, you might need to set the auth middleware or not. To achieve this, you can make use of the dotnet engine's conditional pragmas.

In the ``template.json`` file I declare the symbol ``Authorization`` of type ``bool`` with the default value of true. 

```json
  "Authorization": {
    "type": "parameter",
    "datatype": "bool",
    "defaultValue": "true",
    "description": "Enables the use of authorization with Microsoft.Identity.Web."
  }
```
Which means:
 - If the user sets the value to false, the template engine needs to remove any ``Microsoft.Web.Identity`` reference from the solution.
 - If the user sets this symbol to true, the template engine needs to keep the references to the ``Microsoft.Web.Identity`` library.

In the ``Program.cs`` file the Auth calls are being wrapped by the ``Authorization`` symbol.
```csharp
#if Authorization
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Identity.Web;
#endif

namespace WebApplication
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);

            builder.Services.AddControllers();
            builder.Services.AddEndpointsApiExplorer();
            builder.Services.AddSwaggerGen();
#if Authorization            
            builder.services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddMicrosoftIdentityWebApi(builder.Configuration, "AzureAd");
#endif

            var app = builder.Build();

            if (app.Environment.IsDevelopment())
            {
                app.UseSwagger();
                app.UseSwaggerUI();
            }

            app.UseHttpsRedirection();

#if Authorization
            app.UseAuthentication();
            app.UseAuthorization();
#endif
            app.MapControllers();

            app.Run();
        }
    }
}
```

Same thing in the ``appsettings.json`` file.

```json
﻿{
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

And also in the API Controller

```csharp
#if Authorization
    [Authorize]
#endif
    [ApiVersion("1.0")]
    [ApiController]
    [Route("v{version:apiVersion}/[controller]")]
    [Produces("application/json")]
    public class MyServiceController : ControllerBase
    {
        ...
    }
```

This is a simple example of what you can do with the conditional operators, but there are capable of more things like excluding files based on a symbol or use a symbol value to compute another one.

If you want to read more about it, the following link is the official documentation:
- https://github.com/dotnet/templating/wiki/Conditional-processing-and-comment-syntax

# **Try to build a generic enough dockerfile, and also avoid Container support for .NET SDK**

If you're working with containers, it's beneficial to include a Dockerfile in your application template. Aim to make this Dockerfile generic enough to function seamlessly with most applications created using the template. The goal is to provide a ready-to-use Dockerfile that requires no modification in most of the cases.

Also, if your company uses some custom elements in their images, such as platform images, ensure to configure them in the Dockerfile.

To create a .NET image, you have the option to use Container Support for .NET SDK instead of a Dockerfile.    
However, I try to avoid it, basically because it a niche way of creating images that is not widely known. In contrast, a Dockerfile follows an industry-standard approach that can be adopted by developers from various backgrounds, whether they are .NET developers, Python developers, or SysAdmins.


# **Enable security auditing on your projects**

# **Add a preconfigured nuget.config that uses source mapping**

# **Use something else to store your secrets than the appsettings.json file**

# **Don't be afraid of building your own opinionated NuGets for cross-cutting things like logging, swagger or opentelemtry**

# **Try to avoid opinionated NuGets, go with the industry standards if possible**

# **Add your own .editorconfig**

# **Set langversion to latest**

# **Don't use minimal apis**

# **Always set controller versioning**

# **If possible try to use the default .NET DI implementation, instead of another one**

# **Add an opinionated deploymnet pipeline**

# **Do not use Moq for mocking, use NSubstitute or another one**

# **- Add a README file with the basic sections**



