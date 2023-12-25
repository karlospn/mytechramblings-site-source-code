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

If you want to build a template app


# **Pack all your .NET templates into a single NuGet package**

# **Centralize the NuGet package version using a Directory.Packages.props files**

# **Use a Directory.build.props to group the common properties of every project**

# **Add Visual Studio support**

# **Every project that comprises your app must start with exactly the same name**

# **Make use of the template built-in conditional pragmas to enable or disable certain features**

# **Go for a generic enough dockerfile and avoid Container support for .NET SDK**

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



