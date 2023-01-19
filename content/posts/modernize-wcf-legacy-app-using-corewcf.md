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

The Microsoft recommendation if you want to modernize a WCF app is to migrate to gRPC, but that's not always a viable option. Maybe using SOAP is an inamovible constraint for you ( e.g.  a government services), maybe gRPC doesn't support some obscure feature that WCF does (e.g. Windows integrated authentication), mabye your WCF app is a public service and you can't force your costumers to make the change to another technology like REST or gRPC.   

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
    - BookingMgmt.CoreWCF.WebService
    - BookingMgmt.Contracts
    - BookingMgmt.Application
    - BookingMgmt.Domain
    - BookingMgmt.Database
    - BookingMgmt.SharedKernel
- It also have a couple of unit tests projects and an integration test project:
    - BookingMgmt.Application.UnitTest
    - BookingMgmt.Domain.UnitTest
    - BookingMgmt.CoreWCF.WebService.IntegrationTest

> If you want to take a look at the source code of the app, click [here](https://github.com/karlospn/modernize-wcf-app-using-corewcf/tree/main/before).

The app allow us to manage flight bookings. It allows us to do the following actions:
- Create a new booking.
- Cancel a booking.
- List all active bookings.
- List all canceled bookings.

Now that we know how the app is structured let's start with the modernize process.

# **Migrating the WCF project**

# **Using the .NET Upgrade Assistant tool to migrate the rest of projects**

# **Mending incompatible dependencies**

# **Move configuration settings from using AppSettings to IConfiguration**

# **Manually migrating test projects**

# **Running the app on a Linux container**

# **Wrap-up**