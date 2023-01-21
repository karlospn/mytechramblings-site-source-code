---
title: "How to modernize a legacy .NET Framework WCF app using CoreWCF and .NET 7"
date: 2023-01-17T18:43:16+01:00
draft: true
description: "TBD"
tags: ["dotnet", "wcf", "soap"]
---

> **Just show me the code**   
> As always, if you don’t care about the post I have uploaded the source code on my [Github](https://github.com/karlospn/modernize-wcf-app-using-corewcf).

If you are not old enough, probably you're asking yourself: "What's a WCF?'.
WCF stands for 'Windows Communication Foundation', and if you want to know more about read the official Microsoft docs: [here](https://learn.microsoft.com/es-es/dotnet/framework/wcf/whats-wcf)

Nowadays no one is going to create a WCF from zero, it makes no sense, it's a deprecated technology. But if you're working for a company that's been around long enough and uses .NET as its main stack, it's pretty likely that you are going to stumble with a bunch of legacy applications and I'm pretty sure that some of them are going to be WCF applications.

The WCF framework is not compatible with .NET Core or .NET 7, it only works with .NET Full Framework. 

The Microsoft recommendation if you want to modernize a WCF app is to migrate to gRPC, but that's not always a viable option. Maybe using SOAP is an inamovible constraint for you ( e.g.  a government services), maybe gRPC doesn't support some obscure feature that WCF does (e.g. Windows integrated authentication), maybe your WCF app is a public service and you can't force your costumers to make the change to another technology like REST or gRPC.   

So, what are our options to modernize a WCF app if for some reason you can't move away from the WCF framework? The response to this question is: **CoreWCF**.   

CoreWCF is a port of the service side of Windows Communication Foundation (WCF) to .NET Core. The goal of this project is to enable existing WCF services to move to .NET Core.

**In this post I want to find out how easy (or hard) is try to modernize an .NET Framework WCF app using CoreWCF and .NET 7**

# **CoreWCF**

CoreWCF is a community-driven .NET Foundation project that makes WCF surface area available on modern versions of .NET. It is not a Microsoft-owned project, but Microsoft provides support for CoreWCF. 

CoreWCF isn't a straight port of WCF to .NET - there were a couple of architectural changes that were needed as part of the port:

- Using ASP.NET Core as the service host, push pipeline, and the middleware pattern for extensibility.
- Removing the obsolete Asynchronous Programming Model (APM) programming pattern as it made the codebase incredibly hard to work with, which isn't desirable for a community project wanting to encourage external contributions.
- Removing platform-specific and IO code. Refactoring apps into microservices and Linux-based containers is a common requirement, and so CoreWCF needs to be able to run anywhere that .NET core can be run.

CoreWCF supports many common WCF scenarios, but it does not support all WCF functionality.   
Your modernization experience may vary depending on the overlap between your WCF usage and the functionality included in CoreWCF.

# **Application**

The application we're going to try to modernize has the following features:
- It uses an N-Layer and a Domain-Driven Design architecture.
- It is built entirely using .NET 4.7.2
- It is made by the following projects: 
    - BookingMgmt.WCF.WebService
    - BookingMgmt.Contracts
    - BookingMgmt.Application
    - BookingMgmt.Domain
    - BookingMgmt.Database
    - BookingMgmt.SharedKernel
- It also have a couple of unit tests projects and an integration test project:
    - BookingMgmt.Application.UnitTest
    - BookingMgmt.Domain.UnitTest
    - BookingMgmt.CoreWCF.WebService.IntegrationTest

Here's a look at how it looks in Visual Studio.

<Add-img>

> If you want to take a look at the source code, click [here](https://github.com/karlospn/modernize-wcf-app-using-corewcf/tree/main/before).

But, what this app do? It manages flight bookings. It allows us to do the following actions:
- Create a new booking.
- Cancel a booking.
- List all active bookings.
- List all canceled bookings.

# **Start modernizing the app**

Let's begin defining the end goal for this app. The desired output is the following one:
- Migrate the app from the WCF framework to the CoreWCF framework.
- Upgrade the WCF project from .NET 4.7.2 to .NET 7.
- Upgrade the rest of the app projects (BookingMgmt.Contracts, BookingMgmt.Application, BookingMgmt.Domain and BookingMgmt.SharedKernel) from .NET 4.7.2 to .NET 7 or .NET Standard 2.1.
- Upgrade the test projects from .NET 4.7.2 to .NET 7.
- Run the application on a Linux container.

In the next sections we're going to tackle the tasks listed above.

# **1. Migrating the BookingMgmt.WCF.WebService project**

The first step is to move away from the WCF framework and begin using CoreWCF.

## **Trying to use the .NET Upgrade Assistant tool with the WCF project**

If you read a little bit on the internet about how to do this migration process, you'll see that most of the online posts tells you to use the **[.NET Upgrade Assistant tool](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant)**.   

The .NET Upgrade Assistant tool is a command-line (CLI) tool that helps users interactively upgrade projects from .NET Framework to .NET Standard and .NET 6. 
This tool also is able to assists you in the upgrade of a series of projects types. Currently, the tool supports the following project types:
- ASP.NET MVC
- Windows Forms
- Windows Presentation Foundation (WPF)
- Console
- Libraries
- UWP to Windows App SDK (WinUI)
- Xamarin.Forms to .NET MAUI

The Upgrade Assistant can help you to upgrade WCF projects on .NET Framework to CoreWCF on .NET 6 and later versions of .NET. You can read more about how to do it in the next link:
- https://devblogs.microsoft.com/dotnet/migration-wcf-to-corewcf-upgrade-assistant/

Let's give it a shot.

```bash
$ upgrade-assistant upgrade BookingMgmt.WCF.WebService.csproj
-----------------------------------------------------------------------------------------------------------------
Microsoft .NET Upgrade Assistant v0.4.355802+b2aeae2c0e41fbfed35df6ab2e88b82a0c11be2b

We are interested in your feedback! Please use the following link to open a survey: https://aka.ms/DotNetUASurvey
-----------------------------------------------------------------------------------------------------------------

[17:25:38 ERR] Support for WCF Server-side Services is limited to .NET Full Framework. Consider updating to use CoreWCF (https://aka.ms/CoreWCF/migration) in later provided steps or rewriting to use gRPC (https://aka.ms/migrate-wcf-to-grpc).
[17:25:38 ERR] Project E:\Coding\Dotnet\modernize-wcf-app-using-corewcf\before\BookingMgmt.WCF.WebService\BookingMgmt.WCF.WebService.csproj uses feature(s) that are not supported. If you would like upgrade-assistant to continue anyways please use the "--ignore-unsupported-features" option.
[17:25:38 ERR] Unable to upgrade project E:\Coding\Dotnet\modernize-wcf-app-using-corewcf\before\BookingMgmt.WCF.WebService\BookingMgmt.WCF.WebService.csproj
```

As you can see, the Upgrade Assistant doesn't work from the get-go. The reason why it doesn't work is because the BookingMgmt WCF uses a .svc file and that is not supported.    
The .NET Upgrade Assistant does not support the following scenarios:

❌ WCF server that are Web-hosted and use .svc file.    
❌ Behavior configuration via xml config files except serviceDebug, serviceMetadata, serviceCredentials (clientCertificate, serviceCertificate, userNameAuthentication, and windowsAuthentication).   
❌ Endpoints using bindings other than NetTcpBinding and HTTP-based bindings.   





# **2. Using the .NET Upgrade Assistant tool to migrate the rest of projects**

# **3.Mending incompatible dependencies**

# **4. Move configuration settings from using AppSettings to IConfiguration**

# **5. Manually migrating test projects**

# **6. Running the app on a Linux container**

# **Wrap-up**